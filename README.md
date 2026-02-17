# Self hosting Matrix Synapse on a personal computer
This guide walks through setting up a secure, private Matrix Synapse homeserver on a personal Linux PC. The focus is on a low-resource, privacy-focused configuration with closed registration, a private invite-only Space, external content embedding to minimize storage, automatic 14-day purge for chat and media, and a monthly cleanup script. We use Tailscale Funnel to hide your real IP without port forwarding, a free subdomain from Dynu for a clean URL, rootless Podman with Quadlet for secure container management, Caddy for proxying, and AppArmor for additional security.

## FAQ
1) **What is self-hosting?** Self-hosting means running a service (like Matrix Synapse) on your own computer instead of using someone else's (like Discord's).
2) **Why not use an existing homeserver?** As [this](https://tatsumoto.neocities.org/blog/i-stopped-using-matrix) article sates, home server administrators can impersonate a user or hijack the room/space.
3) **Why self-host?** It gives you full control over your data, privacy, moderation, and customization without relying on third-party providers and dangers like homeserver takeover are low if you don't delegate PL 50+ to untrusted instances.
5) **Why a personal comuter insted of a VPS?** It's free, uses your existing hardware, and is resilient through federation.
6) **What happens when the PC is off?** Your own account appears offline, cannot send/receive messages, read new messages, invite new people or do anything until the PC is back on. Everyone else (people who joined from other homeservers) can keep chatting normally in your Space and rooms even for days or weeks while your PC is off. If no one else from another homeserver has ever joined a particular room that room becomes unreachable until you come back online.
7) **What happens when the PC is back online?** Your account syncs up: you see all messages sent while you were offline. The process usually takes seconds to a few minutes, depending on how long you were offline and how active the rooms were.
8) **You did X wrong!** I'm not an expert, open an issue describing the problem and the solution in detail and I'll be happpy to fix it.
9) **It didn't work for me!** Follow each step carefully, I can't guaranty it will work on every machine.

## What to expect
- Self-hosted Matrix Synapse server on your personal computer
- Real home IP completely hidden (no port forwarding)
- Free, clean subdomain via Dynu
- Rootless Podman + Quadlet containers (systemd-native)
- Secure password handling (Podman secrets)
- Automatic 14-day retention + monthly cleanup

## Minimum requariments
- A modern Linux computer (desktop or laptop)
- At least 4 GB RAM (8+ GB strongly recommended)
- At least 20–50 GB free SSD space
- Stable internet connection
- Ability to run sudo commands
- ~90–120 minutes of time for initial setup

WIP
