# Mii Renderer Enhanced

A self-hosted Mii rendering service. Put in Mii data, get out a PNG/TGA/3D model. Also supports looking up Miis by their Nintendo Network ID.

Built on top of two projects by [ariankordi](https://github.com/ariankordi): the [FFL-Testing renderer](https://github.com/ariankordi/FFL-Testing/tree/renderer-server-prototype) and the [nwf-mii-cemu-toy frontend](https://github.com/ariankordi/nwf-mii-cemu-toy/tree/ffl-renderer-proto-integrate).
```
Browser ──▶ nwf-mii-cemu-toy (Go) ──▶ FFL-Testing (C++)
             :8080, frontend + API        renderer, TCP :12346
                   │                            │
                   ▼                            ▼
              MariaDB (optional)         FFLResHigh.dat
              NNID ↔ Mii lookup         models + textures
```

## Credits

- **[AboodXD](https://github.com/aboood40091)** — [FFL decompilation](https://github.com/aboood40091/ffl) and [RIO framework](https://github.com/aboood40091/rio). This is the core that actually renders Miis.
- **[ariankordi](https://github.com/ariankordi)** — Turned Abood's sample into a TCP renderer, built the frontend/web server, and scraped the NNID database.
- **[HEYimHeroic](https://github.com/HEYimHeroic)** — [Mii data files](https://github.com/HEYimHeroic/MiiDataFiles) you can drop into the renderer.
- **[jaames](https://gist.github.com/jaames)** — [Mii QR code decryption script](https://gist.github.com/jaames/96ce8daa11b61b758b6b0227b55f9f78) used by the frontend.
- **Nintendo** — Created the Mii system and the FFL library. `FFLResHigh.dat` is their property.

## Setup

### Dependencies

```bash
sudo apt update && sudo apt install -y \
  git g++ cmake pkg-config \
  libglfw3-dev zlib1g-dev libgl1-mesa-dev \
  golang-go nodejs npm
```

Ubuntu's Node.js can be old. If `npm` isn't found, install via [NodeSource](https://github.com/nodesource/distributions):

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

### 1. Build the renderer

```bash
git clone --recursive -b renderer-server-prototype https://github.com/ariankordi/FFL-Testing
cd FFL-Testing

# For headless servers (no display/X11):
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_FLAGS="-O3 -march=native" -DRIO_USE_HEADLESS_GLFW=ON
cmake --build build -j$(nproc)
```

On a desktop, drop the `-DRIO_USE_HEADLESS_GLFW=ON`.

The binary ends up at the repo root (`./ffl_testing_2`), not in `build/`. This is intentional.

### 2. Get FFLResHigh.dat

The renderer won't do anything without this file. It has all the 3D models and textures Miis are made of.

From the Miitomo CDN (archived on archive.org):

```bash
cd ~/FFL-Testing
wget -O AFLResHigh.zip "https://web.archive.org/web/20180502054513/http://download-cdn.miitomo.com/native/20180125111639/android/v2/asset_model_character_mii_AFLResHigh_2_3_dat.zip"
unzip AFLResHigh.zip
mv AFLResHigh_2_3.dat FFLResHigh.dat
rm AFLResHigh.zip
```

Or from a Wii U MLC: `sys/title/0005001b/10056000/content/FFLResHigh.dat`

The file must be named exactly `FFLResHigh.dat` and sit next to `ffl_testing_2`. The renderer opens it relative to its working directory.

### 3. Build the frontend

```bash
git clone -b ffl-renderer-proto-integrate https://github.com/ariankordi/nwf-mii-cemu-toy
cd nwf-mii-cemu-toy

npm install && npm run build   # bundle the JS
go build -o ffl-frontend .     # compile the server
```

Key flags for the frontend:

| Flag | Default | What it does |
|---|---|---|
| `-host` | `:8080` | Listen address |
| `-upstream` | `localhost:12346` | Renderer TCP address |
| `-upstreams` | | Comma-separated renderer addresses (round-robin) |
| `-nnid-to-mii-map-db` | | MariaDB connection string for NNID lookup |
| `-cert` / `-key` | | TLS cert/key for HTTPS |
| `-use-x-forwarded-for` | `false` | For reverse proxies |

### 4. Keep it running (PM2)

```bash
sudo npm install -g pm2

# The --cwd flag matters — both programs read files from their working directory
pm2 start /home/YOUR_USER/FFL-Testing/ffl_testing_2 \
  --name ffl-renderer --cwd /home/YOUR_USER/FFL-Testing -- --server

pm2 start /home/YOUR_USER/nwf-mii-cemu-toy/ffl-frontend \
  --name ffl-frontend --cwd /home/YOUR_USER/nwf-mii-cemu-toy \
  -- -upstream localhost:12346 -host :8080

pm2 save && pm2 startup   # survive reboots
```

`pm2 startup` will print a `sudo` command — run it. If things crash, check `pm2 logs <name> --lines 30`. The most common mistake is `exec cwd` pointing to the wrong place — verify with `pm2 describe ffl-renderer | grep cwd`.

### 5. HTTPS

The frontend uses `crypto.subtle` (Miitomo data decryption) and `navigator.clipboard` (copy button). Both require a secure context — HTTPS or localhost. Over plain HTTP at a non-localhost IP, you'll get `Cannot read properties of undefined (reading 'importKey')`.

**Quick fix — self-signed cert:**

```bash
cd ~/nwf-mii-cemu-toy
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout key.pem -out cert.pem

pm2 delete ffl-frontend
pm2 start /home/YOUR_USER/nwf-mii-cemu-toy/ffl-frontend \
  --name ffl-frontend --cwd /home/YOUR_USER/nwf-mii-cemu-toy \
  -- -upstream localhost:12346 -host :8443 \
     -cert /home/YOUR_USER/nwf-mii-cemu-toy/cert.pem \
     -key /home/YOUR_USER/nwf-mii-cemu-toy/key.pem
pm2 save
```

Access `https://YOUR_IP:8443` — accept the certificate warning.

**Proper fix — Nginx reverse proxy with real TLS:**

```nginx
# /etc/nginx/sites-available/mii-renderer
server {
    listen 443 ssl;
    server_name mii.example.com;
    ssl_certificate     /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Then add `-use-x-forwarded-for` to the frontend flags.

### 6. Test it

```bash
# Render test
curl "http://localhost:8080/miis/image.png?data=005057676b565c6278819697bbc3cecad3e6edf301080a122e303a381c235f4a52595c4e51494f585c5f667d848b96&width=512" --output test.png

# NNID lookup (requires database)
curl http://localhost:8080/mii_data/JasmineChlora
```

---

## API

| Endpoint | Description |
|---|---|
| `GET /miis/image.png?data=...&width=512` | Render PNG |
| `GET /miis/image.glb?data=...` | Export binary glTF (.glb) |
| `GET /miis/image.tga?data=...` | Export TGA |
| `GET /mii_data/{nnid}` | NNID lookup (JSON, or binary with `Accept: application/octet-stream`) |
| `GET /mii_data_random` | Random NNID |
| `GET /cmoc_lookup/{id}` | Wii Mii via WiiLink CMOC (`1234-5678-9123`) |
| `GET /miitomo_get_player_data/{id}` | Miitomo/Kaeru player data |

### Render parameters

| Parameter | Default | Description |
|---|---|---|
| `data` | required | Hex or Base64 Mii data (46–96 bytes) |
| `nnid` | | NNID (add `&api_id=1` for Pretendo) |
| `pid` | | Look up by PID |
| `width` | 270 | Resolution (max 4096) |
| `scale` | 2 | SSAA factor |
| `type` | face | `face`, `face_only`, `all_body`, `fflmakeicon`, `ffliconwithbody`, `variableiconbody`, `all_body_sugar` |
| `expression` | normal | Name (`smile`, `anger`, `blink`, `happy`, `frustrated`…) or int. Comma-separated for GLB |
| `shaderType` | wiiu | `wiiu`, `switch`, `miitomo`, `wiiu_blinn`, `ffliconwithbody` |
| `bodyType` | default | `default`, `wiiu`, `switch`, `miitomo`, `fflbodyres`, `3ds` |
| `modelType` | normal | `normal`, `hat`, `face_only` |
| `resourceType` | default | `default`, `middle`, `high`, `very_high`, `low` |
| `bgColor` | transparent | `#RRGGBB` or `RRGGBBAA` |
| `clothesColor` | default | `red`, `orange`, `yellow`, `yellowgreen`, `green`, `blue`, `skyblue`, `pink`, `purple`, `brown`, `white`, `black` |
| `pantsColor` | default | `gray`, `blue`, `red`, `gold`, `body`, `none` |
| `drawStageMode` | all | `opa_only`, `xlu_only`, `mask_only`, `xlu_depth_mask`, `body_only` |
| `splitMode` | none | `front`, `back`, `both` |
| `cameraX/Y/ZRotate` | 0 | Camera rotation |
| `characterX/Y/ZRotate` | 0 | Model rotation |
| `headwearIndex` | -1 | Headwear index |
| `headwearColor` | | Same palette as clothesColor |
| `instanceCount` | 1 | Instances for rotation render |
| `flattenNose` | false | For helmets — include param to enable |
| `lightEnable` | true | Set `0` to disable |
| `texResolution` | =width | Override texture resolution |
| `mipmapEnable` | false | Include to enable |

---

## Troubleshooting

| Problem | Why | Fix |
|---|---|---|
| `FFL_RESULT_ERROR` | Can't find `FFLResHigh.dat` | Check file exists and `pm2 describe ffl-renderer` shows correct `cwd` |
| `bind: address already in use` | Old process still on that port | `sudo fuser -k 8080/tcp` then `pm2 restart ffl-frontend` |
| `importKey` / `writeText` undefined | No secure context (plain HTTP at non-localhost IP) | Set up HTTPS or use localhost |
| NNID returns 404 | No database, or NNID not in it | Add `-nnid-to-mii-map-db` flag; DB only covers NNIDs up to April 2024 |
| MariaDB won't start | Log file size mismatch or version conflict | `sudo rm /mnt/mii-db/ib_logfile*` then restart; or export to SQL first |
| OpenGL errors | Not built headless | Rebuild with `-DRIO_USE_HEADLESS_GLFW=ON`, or use `xvfb-run` |

---

## If the raw datadir doesn't work

Boot MariaDB once with the dump, export it, then import into your default datadir:

```bash
sudo /usr/sbin/mariadbd --datadir=/mnt/mii-db --innodb_log_file_size=256M --innodb_buffer_pool_size=1G &
sleep 5
mariadb-dump -u root miis > ~/miis_dump.sql
sudo killall mariadbd
# Comment out the datadir line in 99-mii.cnf
sudo systemctl start mariadb
sudo mariadb -e "CREATE DATABASE IF NOT EXISTS miis;"
sudo mariadb miis < ~/miis_dump.sql
```

Or use the [pickle dump](https://mega.nz/file/HWJh2BrA#qoJ4Vn_Sdy7b1vWJqjGR9uVvs-yYRQJLaPmu2g2nMdE) with the Python import scripts (slow — ~1000 pickle files).

## Scaling

The renderer is single-threaded — one request at a time. Run multiple instances on different ports and use `-upstreams`:

```bash
pm2 start ... -- -upstreams "localhost:12346,localhost:12347,localhost:12348" -host :8080
```

There are also systemd socket-activated templates (`ffl-testing@.service`, `ffl-testing@.socket`) in the FFL-Testing repo. Build with `-DUSE_SYSTEMD_SOCKET=ON` for those.

Be realistic though — this was patched together from a sample program. A rewrite with multithreading and a built-in HTTP server is in the works.
