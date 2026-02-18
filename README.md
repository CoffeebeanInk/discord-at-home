# Self hosting Matrix Synapse on a user PC
This guide walks through setting up a secure, private Matrix Synapse homeserver on a personal Linux PC. The focus is on a low-resource, privacy-focused configuration with closed registration, a private invite-only Space, external content embedding to minimize storage, automatic 14-day purge for chat and media, and a monthly cleanup script. We use Tailscale Funnel to hide your real IP without port forwarding, a free subdomain from Dynu for a clean URL, rootless Podman with Quadlet for secure container management, Caddy for proxying, and AppArmor for additional security.

## FAQ
1) **What is self-hosting?** Self-hosting means running a service (like Matrix Synapse) on your own hardware instead of using someone else's (like Discord's).
2) **Why self-host?** It gives you full control over your data, privacy, moderation, and customization without relying on third-party providers and dangers like homeserver takeover are low if you don't delegate PL 50+ to untrusted instances.
3) **Why not use an existing homeserver?** As [this](https://tatsumoto.neocities.org/blog/i-stopped-using-matrix) article sates, home server administrators can impersonate a user or hijack the room/space.
5) **Why a personal comuter insted of a VPS?** It's free, uses your existing hardware, and is resilient through federation.
6) **What happens when the PC is off?** Your own account appears offline, cannot send/receive messages, read new messages, invite new people or do anything until the PC is back on. Everyone else (people who joined from other homeservers) can keep chatting normally in your Space and rooms even for days or weeks while your PC is off. If no one else from another homeserver has ever joined a particular room that room becomes unreachable until you come back online.
7) **What happens when the PC is back online?** Your account syncs up: you see all messages sent while you were offline. The process usually takes seconds to a few minutes, depending on how long you were offline and how active the rooms were.
8) **How many resources will this use?**
   
| Scenario                              | RAM              | CPU              | Disk (long-term) | Network          | Power (extra) | Impact on Your PC                  |
|---------------------------------------|------------------|------------------|------------------|------------------|---------------|------------------------------------|
| **Idle / no one chatting**            | 0.6–1.2 GB       | <5%              | 3–8 GB           | <50 KB/s         | 5–10 W        | Unnoticeable                       |
| **Normal chat (10–50 people active)** | 1–2 GB           | 10–40%           | 4–9 GB           | 100 KB–1 MB/s    | 10–20 W       | Barely noticeable                  |
| **After week offline + catch-up**     | 1.5–3.5 GB       | 40–90% (short)   | +0.1–1 GB temp   | 1–5 MB/s         | 20–40 W       | Short CPU spike, fans may spin     |
| **Gaming + streaming at same time**   | +0.5 GB          | +5–10%           | No change        | No change        | +5–15 W       | Negligible (GPU encoding offloads) |


10) **You did X wrong!** I'm not an expert, open an issue describing the problem and the solution in detail and I'll be happpy to fix it.
11) **It didn't work for me!** Follow each step carefully, I can't guaranty it will work on every machine.

## What to expect
- Self-hosted Matrix Synapse server on your personal computer
- Real home IP completely hidden (no port forwarding)
- Free, clean subdomain via Dynu
- Rootless Podman + Quadlet containers (systemd-native)
- Secure password handling (Podman secrets)
- Automatic 14-day retention + monthly cleanup

## Minimum requariments
- A modern Arch based Linux computer (desktop or laptop)
- At least 4 GB RAM (8+ GB strongly recommended)
- At least 20–50 GB free SSD space
- Stable internet connection
- Ability to run sudo commands
- ~90–120 minutes of time for initial setup

---WIP---

#### Step 1: Install Required Packages
Update your system and install core dependencies:
```
sudo pacman -Syu
sudo pacman -S podman postgresql matrix-synapse caddy apparmor tailscale python python-pip git cronie
```
- `matrix-synapse`: Arch's Synapse package (we'll use it for config generation but run in Podman).
- Initialize PostgreSQL: `sudo -u postgres initdb -D /var/lib/postgres/data`, then start and enable: `sudo systemctl enable --now postgresql`.
- Create a Synapse database: `sudo -u postgres psql -c "CREATE USER synapse; CREATE DATABASE synapse OWNER synapse ENCODING UTF8 LC_COLLATE 'C' LC_CTYPE 'C' TEMPLATE template0;"`.

For Python dependencies (for bots and scripts): `pip install matrix-synapse matrix-bot-sdk requests`.

#### Step 2: Set Up Dynu for Dynamic DNS
Dynu provides free dynamic DNS to map your changing home IP to a domain.

1. Sign up at dynu.com and create a hostname (e.g., matrix.example.com).
2. Install a dynamic DNS client: Use `ddclient` (available via AUR: `yay -S ddclient` if using an AUR helper).
3. Configure `/etc/ddclient.conf`:
   ```
   protocol=dyndns2
   use=web, web=checkip.dynu.com/
   server=api.dynu.com
   login=your-dynu-username
   password=your-dynu-api-key
   matrix.example.com
   ```
4. Start and enable: `sudo systemctl enable --now ddclient`.
5. Verify: Check dynu.com dashboard for IP updates every 5-10 minutes.

(If ddclient fails, use Dynu's official Linux client script from their support guides.)

#### Step 3: Set Up Tailscale and Funnel for Secure Exposure
Tailscale creates a secure VPN mesh; Funnel exposes your local service publicly without port forwarding.

1. Install Tailscale: Already done in Step 1.
2. Start and authenticate: `sudo tailscale up --authkey=your-tailscale-auth-key` (get from tailscale.com/admin).
3. Enable MagicDNS and HTTPS in your Tailscale admin console.
4. Enable Funnel: `tailscale funnel --bg` (approves via web; adds "funnel" attribute to your tailnet policy).
5. Expose your future Caddy port (e.g., 443): `tailscale funnel 443` (this creates a public URL like https://your-device.ts.net).
6. Test: Run a temporary server (e.g., `python -m http.server 443`) and access via the Funnel URL.

Funnel handles TLS; traffic proxies securely through Tailscale relays.

#### Step 4: Install and Configure Podman
Podman runs containers rootlessly.

1. Enable user namespaces (for rootless): Edit `/etc/subuid` and `/etc/subgid` to add your user (e.g., `youruser:100000:65536`).
2. Test Podman: `podman run hello-world`.
3. Install podman-compose if needed: `pip install podman-compose` (for easier multi-container setups).

#### Step 5: Configure AppArmor for Podman
AppArmor confines applications for security.

1. Enable AppArmor: `sudo systemctl enable --now apparmor`.
2. For Podman DNS issues (if using bridge networks): Edit `/etc/apparmor.d/local/usr.sbin.dnsmasq`:
   ```
   owner /run/user/[0-9]*/containers/cni/dnsname/*/dnsmasq.conf r,
   owner /run/user/[0-9]*/containers/cni/dnsname/*/addnhosts r,
   ```
3. Reload: `sudo apparmor_parser -R /etc/apparmor.d/usr.sbin.dnsmasq && sudo apparmor_parser /etc/apparmor.d/usr.sbin.dnsmasq`.
4. Apply to Podman: Use `--security-opt apparmor=unconfined` if profiles conflict, but test confined first.

#### Step 6: Install and Configure Caddy as Reverse Proxy
Caddy handles HTTPS and proxies to Synapse.

1. Create `/etc/caddy/Caddyfile`:
   ```
   matrix.example.com {
       reverse_proxy localhost:8008
   }
   ```
   (Adjust port if Synapse uses different; enables auto-HTTPS via Let's Encrypt, but use Tailscale Funnel for home setups to avoid port 80/443 exposure.)
2. Start and enable: `sudo systemctl enable --now caddy`.
3. Test: `curl https://matrix.example.com` (should proxy if Synapse is running).

Integrate with Tailscale: Funnel the Caddy port for external access.

#### Step 7: Set Up Synapse in Podman
Use Podman's Docker compatibility for Synapse's official image.

1. Generate config: `synapse_homeserver --generate-config --server-name matrix.example.com --data-dir /var/lib/synapse --report-stats=no`.
2. Edit `/var/lib/synapse/homeserver.yaml`:
   - Set `database` to PostgreSQL: `database: name: psycopg2, args: {user: synapse, password: yourpass, dbname: synapse, host: localhost}`.
   - Enable federation if needed: `federation_domain_whitelist: null`.
   - Set listeners: `listeners: - port: 8008, bind_addresses: ['::', '0.0.0.0'], type: http, tls: false, resources: - names: [client, federation]`.
3. Run in Podman: `podman run -d --name synapse -v /var/lib/synapse:/data -p 8008:8008 docker.io/matrixdotorg/synapse:latest`.
4. Register admin user: `podman exec -it synapse register_new_matrix_user -c /data/homeserver.yaml http://localhost:8008`.
5. Start: `podman start synapse`.

Proxy through Caddy and expose via Tailscale Funnel.

#### Step 8: Configure 14-Day Chat History Retention
Edit `homeserver.yaml`:
```
retention:
  enabled: true
  default_policy:
    max_lifetime: 14d
  allowed_lifetime_max: 14d
  purge_jobs:
    - longest_max_lifetime: 14d
      interval: 12h
```
Restart Synapse: `podman restart synapse`.

For rooms, admins can set via client: Send `m.room.retention` event with `{"max_lifetime": 1209600000}`.

#### Step 9: Set Up Monthly Cleanup Script
Use a script to purge old data monthly.

1. Clone: `git clone https://github.com/ivansostarko/matrix-synapse-purge-script`.
2. Edit `purge_synapse.sh` for your setup (e.g., retention days=30 for monthly).
3. Make executable: `chmod +x purge_synapse.sh`.
4. Add to crontab (monthly on 1st): `crontab -e` then `0 0 1 * * /path/to/purge_synapse.sh`.
5. It purges events/media >30 days and vacuums PostgreSQL.

For manual purges: Use Synapse admin API (e.g., `POST /_synapse/admin/v1/purge_history/room_id`).

#### Step 10: Set Up Bot to Kick Inactive Users After 30 Days
Create a simple Python bot using matrix-bot-sdk.

1. Create `kick_bot.py`:
   ```python
   import asyncio
   from nio import AsyncClient, RoomMessageText

   HOMESERVER = "https://matrix.example.com"
   USER = "@bot:matrix.example.com"
   PASS = "botpassword"
   ROOM_ID = "!yourroomid:matrix.example.com"  # Replace

   async def main():
       client = AsyncClient(HOMESERVER, USER)
       await client.login(PASS)
       last_active = {}  # Track user activity

       async def message_cb(room, event):
           if room.room_id == ROOM_ID and isinstance(event, RoomMessageText):
               last_active[event.sender] = event.server_timestamp

       client.add_event_callback(message_cb, RoomMessageText)

       # Periodic check (every hour)
       while True:
           await asyncio.sleep(3600)
           now = int(time.time() * 1000)
           for user in last_active:
               if now - last_active.get(user, 0) > 2592000000:  # 30 days ms
                   await client.room_kick(ROOM_ID, user, reason="Inactive for 30 days")

       await client.sync_forever(timeout=30000)

   asyncio.run(main())
   ```
   (Install `nio`: `pip install matrix-nio`.)
2. Register bot user via Synapse admin.
3. Run: `python kick_bot.py` (or as systemd service).
4. Invite bot to room and give mod powers.

(Adapt for multi-room; this is basic—enhance with logging.)

#### Step 11: Install and Use Commet as Matrix Client
Commet is an open-source desktop Matrix client with E2EE, calls, and multi-account support.

1. Download from GitHub: `git clone https://github.com/commetchat/commet`.
2. Build/install: Follow repo instructions (e.g., Flutter-based: `flutter pub get && flutter run`).
3. Or use Flatpak if available: `flatpak install commet`.
4. Launch, log in with your Synapse account (@user:matrix.example.com).
5. Features: Use for chatting, calls, emoji uploads.

Test the setup: Create a room in Commet, invite users, and verify retention/cleanup/bot over time. Monitor logs (`podman logs synapse`) and secure with firewalls (e.g., ufw allow Tailscale). If issues, check Synapse docs or Arch Wiki.
