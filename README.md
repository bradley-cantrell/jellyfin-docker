Self-hosted Jellyfin instance using Docker, Jellyfin, and Caddy. DNS via Cloudflare (with automatic DDNS), reverse proxy via Caddy.

Caddy uses the [caddy-cloudflare-ddns](https://github.com/serfriz/caddy-custom-builds/tree/main/caddy-cloudflare-ddns) image from [serfriz/caddy-custom-builds](https://github.com/serfriz/caddy-custom-builds), which keeps your Cloudflare A/AAAA record(s) updated with your server’s public IP.

Security is the responsibility of the system administrator. Before exposing anything to the internet, ensure you understand the ramifications and secure your system. Linux hardening guides exist for most Linux distros.

# Requirements
1. A (security-hardened) server running Docker with docker compose which will host Jellyfin
2. Cloudflare-managed domain
3. A Cloudflare API token with DNS edit permissions (for the zone)
4. Ports allowed through firewall for Jellyfin host

# Setup
This implementation assumes you own your domain via Cloudflare and have access to create API tokens and edit DNS.

1. **Create a Cloudflare API token** (Dashboard → My Profile → API Tokens → Create Token). Use the “Edit zone DNS” template or a custom token with permission: Zone → DNS → Edit. Limit the token to the zone(s) you use (e.g. `cantrellbl.dev`). Set `CLOUDFLARE_API_TOKEN` in your environment or in a `.env` file in this directory (do not commit `.env`).

2. Allow ports 80 and 443 through your firewall. This varies per firewall and distro. In my case, using `ufw`:

```
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 443/udp
```

3. In the root of this directory, run docker compose:

```docker compose up``` #add the -d flag to run detached, I recommend running without the -d flag first to see logs

Jellyfin and Caddy should start up. At this point, we do not have public access. We need to access the Jellyfin setup page and create an Admin user first. 

4. Jellyfin will output which IP address and port to use to access the setup page, by default localhost:8096. Access that page, either via browser on the local host if it has a GUI or via SSH tunnel on a remote machine if not:

## SSH Tunnel Example
```ssh -L 8096:localhost:8096 username@<jellyfin_host_private_ip>```

then access http://localhost:8096 in browser on the remote machine.

5. Proceed through the Jellyfin setup, creating any user accounts and tweaking any settings. I recommend having one admin user with full privileges in Jellyfin and (at least) one non-admin user for everyday use to limit attack surface.

6. **(Optional)** In the DNS console for your domain, you can add an A record for your subdomain (e.g. `music`) with any placeholder IP—Caddy’s dynamic DNS will update it to your server’s public IP. If you don’t create it, you may need to ensure the zone allows API-created records; the DDNS app can create/update the record. Set `Proxy status` to `Proxied` for Cloudflare’s TLS and DDoS protection. TTL can be left on Auto.

7. Bring Jellyfin and Caddy down:

```docker compose down```

8. Bring Jellyfin and Caddy back up:

```docker compose up```

Watch to ensure both services start successfully. 

9. Access your subdomain to see the Jellyfin login page. This page is now exposed to the internet. Use secure passwords. The caddy-cloudflare-ddns image will keep the A record for your subdomain updated if your public IP changes.

10. Assuming you're happy with your setup, you can bring Jellyfin and Caddy back down once more, then bring them back up detached:

```docker compose up -d```

This will run the containers in the background. Thanks to the `restart: unless-stopped` policy in the docker-compose file, the containers will automatically restart on system reboot.