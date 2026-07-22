# Deploying the backend on Raspberry Pi 5

Runs the Express API always-on via Docker, reachable at `https://am-i-cooked-api.adam-montgomery.ca` through a Cloudflare Tunnel (no port-forwarding, no static IP, no DDNS). The frontend stays on Vercel.

This is the **second** server on the Pi — `moneyball-spring` already occupies `api.adam-montgomery.ca` and host port `8080`. This stack publishes **no host port at all** (the API is only `expose`d on its own compose network, reached by its sidecar `cloudflared`), so the two cannot collide.

## 1. Install Docker

Already done if `moneyball-spring` is running. Otherwise:
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```
Log out and back in for the group change to apply.

## 2. Get the deploy files

No repo clone needed — the image is pulled prebuilt from GHCR. Just grab the two files:
```bash
mkdir am-i-cooked-deploy && cd am-i-cooked-deploy
curl -O https://raw.githubusercontent.com/Aquinnmo/am-i-cooked/main/docker-compose.yml
curl -O https://raw.githubusercontent.com/Aquinnmo/am-i-cooked/main/.env.example
```

## 3. Create the Cloudflare Tunnel

In the Cloudflare dashboard: **Zero Trust → Networks → Tunnels**
1. Create a tunnel (separate from moneyball's), choose the **Docker** connector, copy the token.
2. Add a **Public Hostname**: subdomain `am-i-cooked-api`, domain `adam-montgomery.ca` → Service type `HTTP`, URL `app:5000`.
   (This auto-creates the DNS CNAME — no manual DNS edit needed.)

## 4. Configure secrets

```bash
cp .env.example .env
```
Fill in:
- `TUNNEL_TOKEN` — from step 3.
- `MONGODB_URI` — same Atlas connection string Render was using.
- `GEMINI_API_KEY` — same key Render was using.
- `ALLOWED_ORIGINS` — comma-separated browser origins permitted to call the API. Defaults to the Vercel production URL; add more only if needed.

## 5. Start it

```bash
docker compose pull
docker compose up -d
```
Pulls the prebuilt `arm64` image from GHCR — no compiling on the Pi.

If `docker compose pull` fails with a permission/manifest error, the GHCR package is still private (first push defaults to private). Go to **github.com/Aquinnmo/am-i-cooked → Packages → am-i-cooked → Package settings** and set visibility to Public, or `docker login ghcr.io` on the Pi with a PAT that has `read:packages`.

## 6. Allowlist the Pi in MongoDB Atlas

Atlas **Network Access** was allowing Render's IPs, not the Pi's. Add the Pi's public IP (`curl -s ifconfig.me` on the Pi), or confirm `0.0.0.0/0` is already permitted. Symptom if missed: `/health` returns `"db": false` and logs show a Mongo connection timeout.

## 7. Verify

```bash
curl https://am-i-cooked-api.adam-montgomery.ca/health
```
Should return `{"ok":true,"db":true}`.

CORS check — the second command should print a header, the first should print nothing:
```bash
curl -sI -H "Origin: https://evil.example" https://am-i-cooked-api.adam-montgomery.ca/api/survey/stats | grep -i access-control
curl -sI -H "Origin: https://am-i-cooked-zeta.vercel.app" https://am-i-cooked-api.adam-montgomery.ca/api/survey/stats | grep -i access-control
```

## 8. Point the frontend at it

In **Vercel → project settings → Environment Variables**, set:
```
VITE_API_URL=https://am-i-cooked-api.adam-montgomery.ca
```
then redeploy. Without this variable the frontend falls back to the old Render URL, so removing it is an instant rollback.

## 9. Updating later

```bash
docker compose pull
docker compose up -d
```
Every push to `main` rebuilds the `arm64` image via GitHub Actions (`.github/workflows/docker-publish.yml`); the pull above grabs the new `latest`.

## 10. Boot persistence

`restart: unless-stopped` plus the Docker daemon starting on boot brings the stack back after a reboot or crash. Confirm the daemon is enabled:
```bash
sudo systemctl enable docker
```

## 11. Uptime monitoring

Point UptimeRobot at `https://am-i-cooked-api.adam-montgomery.ca/health`.
