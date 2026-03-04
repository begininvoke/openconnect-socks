# Cisco AnyConnect SOCKS Proxy

Run a **Cisco AnyConnect VPN** via [OpenConnect](https://www.infradead.org/openconnect/) and expose a **SOCKS5** proxy. All traffic through the proxy goes over the VPN tunnel.

Use it to route apps (browsers, CLI, IDE) through your corporate or campus VPN without installing AnyConnect everywhere—one Docker container does the job.

---

## What’s inside the container

| Component    | Role |
|-------------|------|
| **OpenConnect** | Connects to the Cisco AnyConnect VPN |
| **Dante**      | SOCKS5 server on port `1080` |
| **tun2socks**  | Optional TUN device (`tun1`, `198.18.0.1/15`) for transparent proxying |

---

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/) (v2)

---

## Quick start

### 1. Create the env file

Create `env/cisco/.env` with your VPN credentials:

```bash
mkdir -p env/cisco
```

```env
ANYCONNECT_SERVER=vpn.example.com
ANYCONNECT_USER=your_username
ANYCONNECT_PASSWORD=your_password
```

Optional (recommended to avoid cert prompts):

```env
ANYCONNECT_SERVERCERT=sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Get the fingerprint by connecting once without `ANYCONNECT_SERVERCERT` and copying it from the OpenConnect prompt, or from your org’s VPN docs.

**Do not commit** `env/cisco/.env`—it contains secrets. Add `env/cisco/.env` to `.gitignore` if you use git.

### 2. Start the stack

```bash
docker compose up -d --build
```

The VPN usually comes up in **30–60 seconds**. The container stays up and reconnects on failure.

### 3. Check that it works

```bash
# Quick test (VPN must be up first)
curl -x socks5h://127.0.0.1:1080 https://ifconfig.io
```

Or use the helper script (retries for up to ~2.5 minutes):

```bash
./scripts/check-socks.sh
```

If it works, you’ll see the VPN exit IP printed.

---

## Using the proxy

- **Host:** `127.0.0.1`  
- **Port:** `1080`  
- **Protocol:** SOCKS5 (use `socks5h` so DNS is resolved over the VPN)

**Examples:**

```bash
# curl
curl -x socks5h://127.0.0.1:1080 https://internal.company.com

# git (e.g. via GIT_CONFIG)
git config --global http.proxy socks5h://127.0.0.1:1080
git config --global https.proxy socks5h://127.0.0.1:1080
```

In browsers or apps, set the proxy to **SOCKS5** at `127.0.0.1:1080`.

---

## Configuration

### Custom SOCKS port

To expose the proxy on a different host port (e.g. `8080`):

```bash
CISCO_SOCKS_PORT=8080 docker compose -f docker-compose.yml -f docker/docker-compose.publish.yml up -d
```

Then use `127.0.0.1:8080` as the SOCKS proxy.

### Env reference

| Variable | Required | Description |
|----------|----------|-------------|
| `ANYCONNECT_SERVER` | Yes | VPN hostname (e.g. `vpn.example.com`) |
| `ANYCONNECT_USER`   | Yes | VPN username |
| `ANYCONNECT_PASSWORD` | Yes | VPN password |
| `ANYCONNECT_SERVERCERT` | No | Server cert fingerprint (`sha256:...`) to avoid interactive cert prompt |

---

## Troubleshooting

**Proxy doesn’t respond / connection refused**

- Wait 30–60 s after `docker compose up` for the VPN to establish.
- Check logs: `docker compose logs -f vpn-socks`
- Run `./scripts/check-socks.sh` to poll until the proxy is ready.

**Authentication or cert errors**

- Confirm server, user, and password in `env/cisco/.env`.
- If you see a cert prompt, add `ANYCONNECT_SERVERCERT=sha256:...` with the fingerprint shown.

**Container exits or restarts**

- `docker compose logs vpn-socks` for OpenConnect/Dante errors.
- The service uses `restart: unless-stopped`, so it will retry after failures.

---

## Project layout

```
.
├── docker-compose.yml           # Main stack (builds image, runs vpn-socks)
├── docker/
│   ├── docker-compose.publish.yml   # Override for custom SOCKS port
│   └── image/vpn-socks/
│       ├── Dockerfile
│       ├── entrypoint.sh        # VPN + Dante + tun2socks startup
│       ├── sockd.conf           # Dante SOCKS config template
│       └── vpnc-script.sh       # OpenConnect routing script
├── env/cisco/.env               # Your credentials (create this, do not commit)
└── scripts/
    └── check-socks.sh           # Wait for SOCKS and test with curl
```

---

## License

MIT — see [LICENSE.md](LICENSE.md).
