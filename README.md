# Self hosting Matrix Synapse on a user PC
This guide walks through setting up a secure, private Matrix Synapse homeserver on a personal Linux PC. The focus is on a low-resource, privacy-focused configuration with closed registration, a private invite-only Space, external content embedding to minimize storage, automatic 14-day purge for chat and media, and a monthly cleanup script. We use Tailscale Funnel to hide your real IP without port forwarding, a free subdomain from Dynu for a clean URL, rootless Podman with Quadlet for secure container management, Caddy for proxying, and AppArmor for additional security.

## FAQ
1) **What is self-hosting?** Self-hosting means running a service (like Matrix Synapse) on your own hardware instead of using someone else's (like Discord's).
2) **Why self-host?** It gives you full control over your data, privacy, moderation, and customization without relying on third-party providers and dangers like homeserver takeover are low if you don't delegate PL 50+ to untrusted instances.
3) **Why not use an existing homeserver?** As [this](https://tatsumoto.neocities.org/blog/i-stopped-using-matrix) article sates, home server administrators can impersonate a user or hijack the room/space.
5) **Why a personal comuter insted of a VPS?** It's free, uses your existing hardware, and is resilient through federation.
6) **What happens when the PC is off?** Your own account appears offline, cannot send/receive messages, read new messages, invite new people or do anything until the PC is back on. Everyone else (people who joined from other homeservers) can keep chatting normally in your Space and rooms even for days or weeks while your PC is off (this is why you should keep registrations off, only your admin account registred in your homeserver and invite eveyone to join from other homeservers). If no one else from another homeserver has ever joined a particular room that room becomes unreachable until you come back online (you can make an alt account on other homserver and join every room of your Space).
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

---

#### Phase 1: Create a Free Subdomain with Dynu (5 minutes)

1. Go to https://www.dynu.com and sign up for a free account (email and password only, no card needed).  
   **Why?** Dynu provides a reliable, clean subdomain without monthly confirmations or ads, making your server URL professional and easy to share (e.g., for invite links). This is better than a random Tailscale domain for usability.

2. After login, click **Add** → **Hostname**.  
   - Enter a hostname like `chat-yourname` or `matrix-private`.  
   - Choose a free domain from the dropdown (e.g., `dynu.net`).  
   - Full example: `chat-yourname.dynu.net`.  
   - Click **Add** and save default settings.  
   **Why?** This creates a stable address that we will CNAME to Tailscale for IP hiding. It's free forever and supports dynamic updates if needed.

---

#### Phase 2: Install and Configure Tailscale + Funnel (10 minutes)

Tailscale creates a secure VPN mesh and exposes your server publicly without revealing your IP.

1. Install Tailscale:  
   ```bash
   sudo pacman -S tailscale
   sudo systemctl enable --now tailscaled
   tailscale up
   ```  
   **Why?** Tailscale handles NAT traversal and IP hiding. The `up` command generates a login URL — open it in your browser and log in with GitHub/Google/Microsoft (free account). This connects your PC to your private tailnet (e.g., `yourname.ts.net`). It's fast and secure with WireGuard encryption.

2. Enable Funnel:  
   - Go to https://login.tailscale.com/admin/machines.  
   - Find your PC → click the three dots → **Edit route settings**.  
   - Turn **ON** Funnel and HTTPS.  
   - Save.  
   **Why?** Funnel allows public internet access to your local server on port 443 without port forwarding. HTTPS provides free certificates. This is the key to hiding your IP — traffic routes through Tailscale relays.

3. Test Funnel:  
   ```bash
   tailscale funnel 443 http://127.0.0.1:8080
   ```  
   **Why?** This temporarily exposes a local port (we'll use 8080 for Caddy). You will see a public URL like `https://your-pc-name.your-tailnet.ts.net`. This confirms Tailscale is working. Keep it running for now.

---

#### Phase 3: Install Podman, AppArmor, and Caddy (5 minutes)

1. Install packages:  
   ```bash
   sudo pacman -S podman podman-compose caddy apparmor
   sudo systemctl enable --now podman.socket apparmor
   ```  
   **Why?** Podman is daemonless and rootless (more secure than Docker). AppArmor adds mandatory access control to confine containers. Caddy is a simple reverse proxy for HTTPS and .well-known delegation. The `podman.socket` enables rootless mode; AppArmor enforces policies that prevent container escapes.

2. Log out and log back in (or reboot).  
   **Why?** This applies group changes for rootless Podman, allowing containers to run as your user (safer).

---

#### Phase 4: Create Project Directories (2 minutes)

```bash
mkdir -p ~/.config/containers/systemd
mkdir -p ~/matrix-synapse/data/postgres
mkdir -p /etc/caddy/wellknown/matrix
sudo chown -R $USER:$USER /etc/caddy
```

**Why?** These directories hold Quadlet files (for systemd), data volumes, and Caddy configs. The chown ensures Caddy runs as your user for rootless compatibility.

---

#### Phase 5: Create Podman Secret for the Database Password (Secure Storage – 5 minutes)

1. Generate a strong random password:  
   ```bash
   pwgen -s 40 1
   ```  
   **Why?** This creates a secure 40-character random string (e.g., `K7p!vX9mQz$2rT8wL5nYfJ3hB6cA4eD0g...`). Copy it. It's crucial for the Postgres DB; make it unique and long to protect against brute-force if someone gets access.

2. Create the Podman secret:  
   ```bash
   echo -n "K7p!vX9mQz$2rT8wL5nYfJ3hB6cA4eD0g..." | podman secret create pg_password -
   ```  
   **Why?** Podman secrets store the password encrypted (not plain text in files). This protects it from local exploits (e.g., malware reading your home directory). Even if someone reads your configs, they can't see the password.

---

#### Phase 6: Create Quadlet Files (Systemd-Native Containers – 5 minutes)

Quadlet turns containers into native systemd services — auto-start, restart, logging.

1. Postgres Quadlet:  
   ```bash
   nano ~/.config/containers/systemd/matrix-postgres.container
   ```  

   ```ini
   [Unit]
   Description=Matrix PostgreSQL Database
   After=network-online.target

   [Container]
   ContainerName=matrix-postgres
   Image=postgres:16-alpine
   Volume=%h/matrix-synapse/data/postgres:/var/lib/postgresql/data
   Secret=pg_password,target=/run/secrets/pg_password,mode=0400
   Environment=POSTGRES_USER=synapse
   Environment=POSTGRES_DB=synapse
   Environment=POSTGRES_PASSWORD_FILE=/run/secrets/pg_password
   Network=host

   [Service]
   Restart=always

   [Install]
   WantedBy=default.target
   ```

2. Synapse Quadlet:  
   ```bash
   nano ~/.config/containers/systemd/matrix-synapse.container
   ```  

   ```ini
   [Unit]
   Description=Matrix Synapse Homeserver
   After=matrix-postgres.service
   Requires=matrix-postgres.service

   [Container]
   ContainerName=matrix-synapse
   Image=matrixdotorg/synapse:latest
   Volume=%h/matrix-synapse/data:/data
   Environment=SYNAPSE_CONFIG_PATH=/data/homeserver.yaml
   Network=host

   [Service]
   Restart=always

   [Install]
   WantedBy=default.target
   ```

**Why Quadlet?** It's the cleanest, most secure way to run containers as system services — rootless, declarative, auto-restarts, integrates with systemd for logging/resource limits. Seccomp is built-in (filters dangerous calls); AppArmor (enabled in Phase 3) confines the containers further.

---

#### Phase 7: Generate Synapse Config (5 minutes)

1. Reload systemd:  
   ```bash
   systemctl --user daemon-reload
   ```  
   **Why?** This loads the Quadlet files so Podman can create the services.

2. Start Postgres:  
   ```bash
   systemctl --user start matrix-postgres
   ```  
   **Why?** Postgres must be running to generate Synapse config.

3. Generate config:  
   ```bash
   podman run --rm -v ~/matrix-synapse/data:/data \
     -e SYNAPSE_SERVER_NAME=chat-yourname.dynu.net \
     -e SYNAPSE_REPORT_STATS=no \
     matrixdotorg/synapse:latest generate
   ```  
   **Why?** This creates the initial `homeserver.yaml` with your domain.

4. Edit the config:  
   ```bash
   nano ~/matrix-synapse/data/homeserver.yaml
   ```  

   Replace these sections:

   ```yaml
   server_name: "chat-yourname.dynu.net"
   public_baseurl: "https://chat-yourname.dynu.net"

   listeners:
     - port: 8008
       tls: false
       type: http
       x_forwarded: true
       bind_addresses: ['127.0.0.1']
       resources:
         - names: [client, federation]

   retention:
     enabled: true
     default_policy:
       max_lifetime: 1209600000   # 14 days

   media_retention:
     local_media_lifetime: 14d
     remote_media_lifetime: 14d

   max_upload_size: 10M

   database:
     name: psycopg2
     args:
       user: synapse
       password_file: /run/secrets/pg_password
       database: synapse
       host: localhost
   ```

**Why?** This sets your domain, enables purge, and uses the secret password file for security.

---

#### Phase 8: Caddy Configuration (Tailscale Port Workaround – 5 minutes)

1. Create Caddyfile:  
   ```bash
   nano /etc/caddy/Caddyfile
   ```  

   ```caddy
   chat-yourname.dynu.net {
       reverse_proxy localhost:8008
   }

   chat-yourname.dynu.net/.well-known/matrix/* {
       root * /etc/caddy/wellknown
       file_server
   }
   ```  

2. Create the well-known files:  
   ```bash
   nano /etc/caddy/wellknown/matrix/client
   ```  

   ```json
   {
     "m.homeserver": { "base_url": "https://chat-yourname.dynu.net" }
   }
   ```  

   ```bash
   nano /etc/caddy/wellknown/matrix/server
   ```  

   ```json
   {
     "m.server": "chat-yourname.dynu.net:443"
   }
   ```  

3. Restart Caddy:  
   ```bash
   sudo systemctl restart caddy
   ```  

**Why Caddy?** Tailscale Funnel only supports 443 → Caddy proxies it to Synapse on 8008 and serves .well-known files for federation discovery. This is the key workaround for port limits.

---

#### Phase 9: Enable & Start Everything

```bash
systemctl --user daemon-reload
systemctl --user enable --now matrix-postgres matrix-synapse
sudo systemctl restart caddy
```

**Why?** This starts the services and enables auto-start on login/boot.

---

#### Phase 10: Create Admin Account (2 minutes)

```bash
podman exec -it matrix-synapse register_new_matrix_user http://localhost:8008 -c /data/homeserver.yaml -a @yourusername:chat-yourname.dynu.net
```

**Why?** This creates your single local admin account with the -a flag for admin privileges.

---

#### Phase 11: Tailscale Funnel as Systemd Service (2 minutes)

```bash
sudo tee /etc/systemd/system/tailscale-funnel.service <<EOF
[Unit]
Description=Tailscale Funnel for Matrix
After=network.target tailscaled.service

[Service]
ExecStart=/usr/bin/tailscale funnel 443 http://127.0.0.1:8080
Restart=always
User=$USER

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now tailscale-funnel
```

**Why?** This makes Funnel start automatically and restart if it crashes.

---

#### Phase 12: Monthly Cleanup Script (5 minutes)

```bash
nano ~/matrix-synapse/purge.sh
```

```bash
#!/bin/bash
BEFORE=$(date -d '60 days ago' +%s000)

curl -X POST "http://localhost:8008/_synapse/admin/v1/purge_media_cache?before_ts=$BEFORE" \
  -H "Authorization: Bearer YOUR_ADMIN_TOKEN_HERE"

find ~/matrix-synapse/data/media_store -type d -empty -delete
```

**Why?** This purges old media cache (beyond the 14-day automatic) and cleans empty directories. Replace `YOUR_ADMIN_TOKEN_HERE` with your real token (get from client devtools: F12 → Network → any request → copy Authorization header).

```bash
chmod +x ~/matrix-synapse/purge.sh
crontab -e
```

Add:

```bash
0 12 1 * * ~/matrix-synapse/purge.sh >> ~/matrix-purge.log 2>&1
```

**Why?** Runs monthly at 12:00 PM — keeps storage clean automatically.

---

#### Phase 13: Matrix Inactive Member Kicker Bot (15 minutes)

```python
#!/usr/bin/env python3
"""
Matrix Room Inactive Member Kicker Bot
Kicks users inactive for more than 30 days from a room/Space.
Works for users from ANY homeserver.
"""

import asyncio
import time
from datetime import datetime, timedelta
from nio import AsyncClient, LoginResponse, RoomMember

# ────────────────────────────────────────────────
#  CONFIG – CHANGE THESE VALUES
# ────────────────────────────────────────────────

HOMESERVER      = "https://chat-yourname.dynu.net"      # Your homeserver URL
BOT_USER        = "@bot-inactive-kicker:chat-yourname.dynu.net"  # Bot's MXID
BOT_PASSWORD    = "your-bot-password-here"               # Bot's password
ROOM_ID         = "!xxxxxxxxxxxxxxxxxxxxxxxx:chat-yourname.dynu.net"  # Room or Space ID (get from client settings)
INACTIVE_DAYS   = 30
DRY_RUN         = True                                   # Set to False to actually kick

# ────────────────────────────────────────────────

async def main():
    client = AsyncClient(HOMESERVER, BOT_USER)

    # Login
    resp = await client.login(password=BOT_PASSWORD)
    if not isinstance(resp, LoginResponse):
        print("Login failed:", resp)
        return

    print(f"Logged in as {BOT_USER}")

    # Get current room members
    members_resp = await client.room_members(ROOM_ID)
    if not members_resp:
        print("Failed to get room members")
        return

    now = int(time.time() * 1000)  # milliseconds
    cutoff_ts = now - (INACTIVE_DAYS * 24 * 60 * 60 * 1000)

    kicked = 0
    for member in members_resp.members:
        mxid = member.user_id
        if mxid == BOT_USER:
            continue  # don't kick self

        # Get member's last active timestamp
        last_active_ms = member.last_active_ago or 0
        last_seen_ts = now - last_active_ms if last_active_ms else 0

        if last_seen_ts < cutoff_ts:
            reason = f"Inactive for more than {INACTIVE_DAYS} days (last seen {datetime.fromtimestamp(last_seen_ts/1000)})"
            print(f"Would kick {mxid} – {reason}")

            if not DRY_RUN:
                kick_resp = await client.room_kick(ROOM_ID, mxid, reason)
                if kick_resp.is_success():
                    print(f"Kicked {mxid}")
                    kicked += 1
                else:
                    print(f"Failed to kick {mxid}: {kick_resp}")
            else:
                kicked += 1  # count for dry-run

    print(f"\nTotal inactive users found: {kicked}")
    if DRY_RUN:
        print("Dry run – no kicks performed. Set DRY_RUN = False to actually kick.")

    await client.close()

if __name__ == "__main__":
    asyncio.run(main())
```

1. Install Dependencies
   Open your terminal and run:  
   ```bash
   pip install --user matrix-nio aiohttp
   ```  
   This installs the Python libraries needed for the bot to connect to Matrix.

2. Create the Bot Account
   In your terminal, run:  
   ```bash
   podman exec -it matrix-synapse register_new_matrix_user http://localhost:8008 -c /data/homeserver.yaml @bot-inactive-kicker:chat-yourname.dynu.net
   ```  
   Follow the prompts to set a strong password. Note it down.

3. Give the Bot Power in the Room 
   - Log into your Matrix client (Commet or Cinny) with your admin account.  
   - Go to the room or Space you want to monitor.  
   - Click the room name → Settings → Roles & Permissions.  
   - Invite `@bot-inactive-kicker:chat-yourname.dynu.net`.  
   - Set its power level to **50** (moderator level).  
   - Save.

4. Save the Script
   In terminal:  
   ```bash
   nano ~/matrix-synapse/kick-inactive-bot.py
   ```  
   Paste the script above. Replace the CONFIG section with your values:  
   - HOMESERVER = your server's URL  
   - BOT_USER = the bot's MXID  
   - BOT_PASSWORD = the bot's password  
   - ROOM_ID = the room's ID (get from client: Room settings → Advanced → Room ID)  
   - Set DRY_RUN = True for testing.  
   Save (Ctrl+O → Enter → Ctrl+X).

5. Make it Executable
   ```bash
   chmod +x ~/matrix-synapse/kick-inactive-bot.py
   ```

6. Test the Bot Manually
   ```bash
   python3 ~/matrix-synapse/kick-inactive-bot.py
   ```  
   - It should log in and print "Would kick [user]..." for inactive users (no actual kicks in dry run).  
   - If it works, set DRY_RUN = False to enable real kicks.

7. Run It Automatically Every Month
   - Run:  
     ```bash
     crontab -e
     ```  
   - Add this line at the end:  
     ```bash
     0 12 1 * * python3 ~/matrix-synapse/kick-inactive-bot.py >> ~/kick-log.txt 2>&1
     ```  
     - Save and exit.  
   - This runs the bot monthly at 12:00 PM. Logs go to kick-log.txt in your home folder.

**Why?** This bot automatically kicks users from a room who have been inactive for 30 days or more. It works for users from any homeserver (local or federated). The bot needs to be a member of the room with power level 50 or higher (to have kick permission).

---

#### Phase 14: Create Your Space & Rooms (5 minutes)

Log in at `https://chat-yourname.dynu.net` (using Commet or Cinny).

1. Create a **Space** → **Private (Invite Only)**  
2. Add rooms:
   - Main Chat → Join rule = **Space members**

**Why?** This creates a tiered, private community with restricted access.

---

#### Phase 15: Dynu CNAME to Tailscale (2 minutes)

In Tailscale admin → **DNS** → **Add DNS record**  
- Type: **CNAME**
- Name: `chat-yourname`
- Target: `your-pc-name.your-tailnet.ts.net`

**Why?** This aliases the Dynu domain to Tailscale, giving you a clean URL while hiding your IP.

---

#### Phase 16: Final Test

- Go to https://federationtester.matrix.org → enter `chat-yourname.dynu.net` → should pass.  
- Join from a matrix.org account.  
- Share the Space invite link.

---

You now have a secure, private, self-hosted Matrix server.
