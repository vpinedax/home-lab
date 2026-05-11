# Secure Remote Access for Self-Hosted Media Server

Remote access architecture for self-hosted services using a cloud VPS, Tailscale, and Nginx Proxy Manager. This setup allows external users to access services hosted on a home network without exposing the home network to the public internet.

## Project Goals

- Enable family members to access a self-hosted Jellyfin media server from anywhere
- Avoid exposing the home network or home IP to the public internet
- Use a real domain with valid TLS certificates so my family and I will not have to download tailscale + apps. 
- Build a reusable architecture that can serve additional services (e.g. Navidrome, Immich) with minimal additional work

## Architecture

```
End User Device (remote)
        │
        │  https://jellyfin.vpinedafam.fyi
        ▼
Cloudflare DNS (resolves to VPS public IP)
        │
        ▼
Hetzner Cloud VPS (Hillsboro, OR)
   ├─ Nginx Proxy Manager (Docker)
   │   - TLS termination (Let's Encrypt)
   │   - Reverse proxy
   └─ Tailscale client
        │
        │  Encrypted Tailscale tunnel
        ▼
UGREEN DXP2800 NAS (home)
   ├─ Tailscale client (Docker)
   └─ Jellyfin (Docker, port 8899)
```

## Component Summary

| Component | Role | Cost |
|---|---|---|
| Hetzner Cloud VPS (CPX11) | Public-facing reverse proxy host | ~$7/mo |
| Domain (`vpinedafam.fyi` via Porkbun) | Public hostname | ~$15/yr |
| Cloudflare | DNS management | Free |
| Tailscale | Encrypted mesh VPN between VPS and NAS | Free (personal) |
| Nginx Proxy Manager | Reverse proxy with GUI and Let's Encrypt automation | Free |
| Jellyfin | Self-hosted media server on the NAS | Free |

## Hardware Inventory

- **VPS:** Hetzner CPX11 (2 vCPU, 2 GB RAM, Ubuntu 22.04, Hillsboro OR region)
- **NAS:** UGREEN DXP2800 running UGOS, Jellyfin installed via the UGOS app store (managed as a Docker container under the hood)

## Setup Walkthrough

### 1. Provision the cloud VPS

Created a Hetzner Cloud project and provisioned a CPX11 instance in the `us-west` (Hillsboro) location running Ubuntu 22.04. An SSH key pair was generated locally with `ssh-keygen -t ed25519` and the public key was added to the server at creation time.

After provisioning, the system was updated:

```bash
ssh root@<vps-public-ip>
apt update && apt upgrade -y
```

### 2. Buy a domain and configure DNS

Purchased `vpinedafam.fyi` via Porkbun with WHOIS privacy enabled. Then:

1. Created a free Cloudflare account
2. Added `vpinedafam.fyi` as a site on the Free plan
3. Replaced Porkbun's default nameservers with the Cloudflare-assigned nameservers
4. Once Cloudflare reported the domain as Active, created an A record:
   - **Name:** `jellyfin`
   - **Type:** `A`
   - **Value:** Hetzner VPS public IPv4
   - **Proxy status:** DNS only (gray cloud) — Cloudflare's proxy disallows heavy video streaming per their ToS

### 3. Install Docker and Tailscale on the VPS

```bash
# Docker
curl -fsSL https://get.docker.com | sh

# Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

The `tailscale up` command outputs an auth URL; opening it in a browser and signing in associates the VPS with a Tailscale account (Tailnet). The VPS receives a `100.x.x.x` Tailscale IP, retrievable via:

```bash
tailscale ip -4
```

### 4. Install Tailscale on the NAS

The UGREEN DXP2800 runs UGOS, a Linux-based NAS OS that exposes Docker. After enabling SSH in UGOS settings, Tailscale was installed as a Docker container alongside Jellyfin:

```bash
sudo docker run -d \
  --name=tailscale \
  --hostname=ugreen-nas \
  --restart=unless-stopped \
  --network=host \
  --cap-add=NET_ADMIN \
  --cap-add=NET_RAW \
  -v ~/tailscale:/var/lib/tailscale \
  -v /dev/net/tun:/dev/net/tun \
  tailscale/tailscale:latest \
  tailscaled
```

Authenticated the container to the same Tailnet:

```bash
sudo docker exec tailscale tailscale up
```

Confirmed connectivity between the VPS and NAS:

```bash
# From the VPS
tailscale ping <nas-tailscale-ip>
```

### 5. Deploy Nginx Proxy Manager on the VPS

Created a Docker Compose stack at `~/npm/docker-compose.yml`:

```yaml
services:
  npm:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: npm
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

Started the stack:

```bash
cd ~/npm
docker compose up -d
```

Logged into NPM at `http://<vps-public-ip>:81` with the default credentials (`admin@example.com` / `changeme`), then immediately set a strong admin email/password and enabled 2FA.

### 6. Configure the proxy host

In NPM, added a new Proxy Host:

- **Domain Names:** `jellyfin.vpinedafam.fyi`
- **Scheme:** `http`
- **Forward Hostname / IP:** NAS's Tailscale IP (`100.x.x.x`)
- **Forward Port:** `8899` (the port Jellyfin actually listens on for this UGOS install)
- **Block Common Exploits:** enabled
- **Websockets Support:** enabled

On the SSL tab:

- **SSL Certificate:** Request a new SSL Certificate (Let's Encrypt)
- **Force SSL:** enabled
- **HTTP/2 Support:** enabled

Saved the host and waited for the Let's Encrypt cert to issue. Tested by visiting `https://jellyfin.vpinedafam.fyi` from an external network — the Jellyfin login page loaded over HTTPS with a valid certificate.

## Troubleshooting Encountered

- **502 Bad Gateway on first test.** NPM was forwarding to port 8096 (the standard Jellyfin default), but the UGOS-managed Jellyfin container was bound to port 8899. Updating the forward port in the proxy host fixed it. `curl -I http://<nas-tailscale-ip>:8096` from the VPS returned "Connection refused", confirming the issue was a port mismatch and not a tunnel problem.

- **SSH permission denied to the NAS.** The UGOS account username is case-sensitive — using the wrong capitalization caused repeated auth failures.

- **`docker run` permission denied on the NAS.** UGOS requires `sudo` for Docker commands by default. `docker --version` worked because it doesn't talk to the daemon, but `docker run` does. Resolved by prefixing commands with `sudo`.

## Security Notes

- The home network has **no inbound ports forwarded**. All inbound traffic to home services flows through the encrypted Tailscale tunnel from the VPS.
- The home IP is never exposed publicly — only the Hetzner VPS IP is reachable from the internet.
- TLS is enforced end-to-user via Let's Encrypt.
- NPM admin panel is protected with a strong password plus 2FA.
- Jellyfin login attempt limits and non-admin user accounts are recommended for end users.

## Extending the Setup

This architecture is designed to be reused for any self-hosted service on the NAS. To add a new service (e.g. Navidrome for music, Immich for photos):

1. Deploy the service on the NAS, listening on some port (e.g. Navidrome on `4533`)
2. In Cloudflare, add a new A record (e.g. `music.vpinedafam.fyi`) pointing to the same Hetzner VPS public IP
3. In NPM, add a new Proxy Host pointing `music.vpinedafam.fyi` to `<nas-tailscale-ip>:4533` with a fresh Let's Encrypt cert

No additional VPS configuration, port forwarding, or DNS infrastructure is needed.

## Relevant Skills Demonstrated

- Cloud infrastructure provisioning (Hetzner Cloud)
- Linux system administration (Ubuntu, SSH key auth, package management)
- Docker and Docker Compose for service deployment
- Reverse proxy configuration with TLS termination
- Mesh VPN networking with Tailscale (WireGuard-based)
- DNS management and TLS certificate automation (Let's Encrypt via ACME)
- Defense-in-depth: no exposed home IP, encrypted transport, hardened admin access

## Future Improvements

- Rebuild the VPS provisioning step with Terraform (infrastructure as code)
- Manage NPM and supporting containers with an Ansible playbook
- Encrypt the Cloudflare API token and Tailscale auth key with Ansible Vault
- Add monitoring (Uptime Kuma) for the proxy and downstream services
- Migrate the reverse proxy from Nginx Proxy Manager to Caddy for declarative configuration
