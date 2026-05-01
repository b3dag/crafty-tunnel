# crafty + cloudflare tunnel

A self-hosted Minecraft server manager (Crafty Controller) exposed publicly through a Cloudflare Tunnel — no open ports, no port forwarding, free TLS.

This repo is the deployment recipe. Crafty itself is a separate project — see [crafty-controller/crafty-4](https://gitlab.com/crafty-controller/crafty-4).

---

## What this gets you

- Crafty running on a fresh Debian box at `https://crafty.YOUR_DOMAIN.com`
- Cloudflare-issued TLS certificate (no Let's Encrypt, no certbot)
- Nothing publicly exposed on your home/server network — Cloudflare reaches Crafty via outbound tunnel
- Clean systemd setup for both Crafty and the tunnel
- Everything uses default install locations — no custom paths to remember

---

## Prerequisites

- A fresh Debian 12 or 13 box (works on Ubuntu 22.04+ as well)
- A domain you control, with DNS managed in Cloudflare
- A Cloudflare account
- Sudo access on the server

---

## Table of contents

1. [Install Crafty](#1-install-crafty)
2. [Install cloudflared](#2-install-cloudflared)
3. [Authenticate and create the tunnel](#3-authenticate-and-create-the-tunnel)
4. [Configure the tunnel](#4-configure-the-tunnel)
5. [Run cloudflared as a service](#5-run-cloudflared-as-a-service)
6. [Open the dashboard](#6-open-the-dashboard)
7. [Day-to-day use](#day-to-day-use)
8. [Troubleshooting](#troubleshooting)
9. [Security notes](#security-notes)

---

## 1. Install Crafty

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl

git clone https://gitlab.com/crafty-controller/crafty-installer-4.0.git
cd crafty-installer-4.0
sudo ./install_crafty.sh
```

The installer is interactive. **Accept the default install path** (`/var/opt/minecraft/crafty`) when prompted — the rest of this README assumes the default location.

The installer creates a `crafty` user and sets up a systemd service.

Enable and start Crafty (the installer doesn't always start it on its own):

```bash
sudo systemctl enable crafty
sudo systemctl start crafty
sudo systemctl status crafty
```

You should see `active (running)`. Also confirm it's actually listening:

```bash
ss -tlnp | grep 8443
```

Crafty should be listening on `0.0.0.0:8443`. The web UI is reachable at `https://<server-ip>:8443` from inside your LAN if you want to test it locally before setting up the tunnel.

Crafty's first-run admin password is in:

```bash
sudo cat /var/opt/minecraft/crafty/crafty-4/app/config/default-creds.txt
```

Read it once, then change the password from the UI after logging in.

---

## 2. Install cloudflared

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list

sudo apt update
sudo apt install -y cloudflared

cloudflared --version
```

---

## 3. Authenticate and create the tunnel

cloudflared needs to authenticate to your Cloudflare account once. We'll run this as root since the tunnel will run as a system service:

```bash
sudo cloudflared tunnel login
```

This prints a Cloudflare URL. Open it in any browser, log in, and pick the domain you want to use. The credentials get saved to `/root/.cloudflared/cert.pem`.

Now create the tunnel itself:

```bash
sudo cloudflared tunnel create crafty
```

It prints two things:
- A **tunnel UUID** like `abc12345-6789-...`
- A **credentials JSON** path like `/root/.cloudflared/abc12345-6789-...json`

**Note the UUID — you'll use it in the next step.** The credentials file stays where it was created.

Route DNS for your subdomain to this tunnel:

```bash
sudo cloudflared tunnel route dns crafty crafty.YOUR_DOMAIN.com
```

That creates a CNAME record in Cloudflare DNS automatically.

---

## 4. Configure the tunnel

Pull this repo somewhere temporary (or download `config.yml.example` directly) and put the config in cloudflared's default location:

```bash
git clone https://github.com/b3dag/craftytunnel /tmp/craftytunnel
sudo cp /tmp/craftytunnel/config.yml.example /root/.cloudflared/config.yml
sudo nano /root/.cloudflared/config.yml
```

Replace:
- `<YOUR_TUNNEL_UUID>` with the UUID from step 3 (in **two places** — `tunnel:` and `credentials-file:`)
- `crafty.YOUR_DOMAIN.com` with your real subdomain

The result should look like this:

```yaml
tunnel: abc12345-6789-...
credentials-file: /root/.cloudflared/abc12345-6789-....json

ingress:
  - hostname: crafty.YOUR_DOMAIN.com
    service: https://127.0.0.1:8443
    originRequest:
      noTLSVerify: true
  - service: http_status:404
```

Why `noTLSVerify: true`? Crafty serves HTTPS using a self-signed certificate on `127.0.0.1:8443`. Without this flag, cloudflared would refuse the connection because the cert isn't trusted. Since both ends are on the same machine and never see the network, this is safe.

Test the tunnel runs:

```bash
sudo cloudflared tunnel run crafty
```

If everything's right, you'll see `Connection ... registered with protocol: quic`. Press Ctrl+C — we'll run it as a service next.

---

## 5. Run cloudflared as a service

cloudflared has a built-in command to install itself as a system service using the config you just created:

```bash
sudo cloudflared service install
sudo systemctl enable --now cloudflared
sudo systemctl status cloudflared
```

Should say `active (running)`. Confirm with the logs:

```bash
sudo journalctl -u cloudflared --no-pager -n 20
```

You should see `Connection ... registered`. The tunnel is live and will start automatically on boot.

---

## 6. Open the dashboard

Visit `https://crafty.YOUR_DOMAIN.com` in your browser. Crafty's login page should load with a valid TLS certificate (from Cloudflare).

Log in with the credentials from `default-creds.txt` (see step 1) and **change the password immediately**.

You're done.

---

## Day-to-day use

### Adding a Minecraft server

Crafty's UI handles everything — `Servers → Create New Server` walks you through downloading a vanilla, Fabric, Forge, Paper, or modded jar, setting RAM, accepting the EULA, and starting it.

Servers Crafty creates listen on regular Minecraft ports (default `25565`). To make them publicly reachable for players, you have a few options:

- **playit.gg** for free public IPs without port forwarding
- **Cloudflare Spectrum** (paid) for TCP routing through Cloudflare
- **Port forwarding** on your router if you have a public IPv4

The Cloudflare Tunnel in this repo handles the **Crafty web UI** but not the Minecraft TCP traffic itself. Cloudflare's free tunnel doesn't proxy raw TCP for game protocols.

### Updating cloudflared

```bash
sudo apt update && sudo apt upgrade -y
sudo systemctl restart cloudflared
```

### Updating Crafty

Crafty's update process is documented in their [wiki](https://gitlab.com/crafty-controller/crafty-4/-/wikis/home).

### Restarting things

```bash
sudo systemctl restart crafty             # Crafty itself
sudo systemctl restart cloudflared        # the tunnel
```

### Editing the tunnel config later

```bash
sudo nano /root/.cloudflared/config.yml
sudo systemctl restart cloudflared
```

---

## Troubleshooting

### `https://crafty.YOUR_DOMAIN.com` shows a 502 or "tunnel error" page

The tunnel is reaching cloudflared, but cloudflared can't reach Crafty.

```bash
# Is Crafty actually listening on 8443?
ss -tlnp | grep 8443
sudo systemctl status crafty
```

If Crafty isn't running, start it: `sudo systemctl start crafty`.

### Tunnel won't start

```bash
sudo journalctl -u cloudflared --no-pager -n 50
```

| Error | Fix |
| --- | --- |
| `Tunnel credentials file '<path>' doesn't exist` | The `credentials-file` path in `config.yml` doesn't match the actual file. Run `sudo ls /root/.cloudflared/*.json` to see the real path. |
| `<YOUR_TUNNEL_UUID>` placeholder still in config | You forgot to replace it. Edit `/root/.cloudflared/config.yml` and put the real UUID in both places. |
| `Cannot determine default origin certificate` | You didn't run `sudo cloudflared tunnel login`, or `/root/.cloudflared/cert.pem` got deleted. |

### TLS error / handshake failed in cloudflared logs

You forgot `noTLSVerify: true` in the ingress config, or you used `service: http://...` instead of `https://...`. Crafty serves HTTPS, not HTTP.

### "Bad gateway" only for some requests

Crafty might be slow on first response after idle. Try refreshing. If it's persistent, check Crafty's logs:

```bash
sudo journalctl -u crafty --no-pager -n 100
```

---

## Security notes

- **Crafty's local cert is self-signed.** That's fine — only cloudflared on the same machine talks to it. The connection from your browser to Cloudflare is still proper TLS.
- **`noTLSVerify: true` only applies to localhost.** Never use it for ingress entries that reach over the network.
- **Crafty serves on `0.0.0.0:8443` by default.** This means it's also reachable from your LAN. If you want to lock it down to localhost only, edit Crafty's config (`/var/opt/minecraft/crafty/crafty-4/app/config/config.json`) and change the bind address to `127.0.0.1`. Then the tunnel is the only way in.
- **Cloudflare can decrypt your traffic.** Standard tradeoff with any Cloudflare-proxied service. Fine for a personal Minecraft panel; reconsider for anything sensitive.
- **Change Crafty's default admin password on first login.**
- **Cloudflare Tunnel credentials (`/root/.cloudflared/*.json`) are sensitive.** Treat them like SSH keys. Don't commit them to git (the included `.gitignore` blocks `*.json`).
- **Consider Cloudflare Access** if you want a real auth layer in front of Crafty (Google/Github login, MFA, IP allowlisting, etc.). It's free up to 50 users.

---

## File locations

| What | Where |
| --- | --- |
| Crafty install | `/var/opt/minecraft/crafty/crafty-4/` |
| Crafty default credentials | `/var/opt/minecraft/crafty/crafty-4/app/config/default-creds.txt` |
| Crafty systemd unit | `/etc/systemd/system/crafty.service` (installed by Crafty's installer) |
| Tunnel config | `/root/.cloudflared/config.yml` |
| Tunnel credentials | `/root/.cloudflared/<UUID>.json` |
| Tunnel auth cert | `/root/.cloudflared/cert.pem` |
| Tunnel systemd unit | `/etc/systemd/system/cloudflared.service` (installed by `cloudflared service install`) |

---

## What's in this repo

- `README.md` — this guide
- `config.yml.example` — template for `/root/.cloudflared/config.yml`

That's it. Everything else uses tool defaults — Crafty's installer creates its own systemd unit, and `cloudflared service install` creates its own.

---

## License

Personal project. Adapt freely.
