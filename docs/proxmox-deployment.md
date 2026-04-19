# Deploying ClaudeClaw on a Linux VM in Proxmox

This guide walks you through deploying ClaudeClaw on a Linux virtual machine (VM) running on a Proxmox server. It assumes you have a Proxmox host already set up and that you can log into the Proxmox web UI (usually `https://your-proxmox-ip:8006`).

By the end of this guide, you'll have ClaudeClaw running 24/7 as a background service on a dedicated VM, restarting automatically on reboot, with optional dashboard access from your phone.

> **Time budget**: about 45-60 minutes start to finish, mostly waiting on downloads.

---

## Why a VM (and not an LXC container)?

You can technically run ClaudeClaw inside an LXC container, but a full VM is the safer choice for beginners:

- The Claude Code CLI sometimes spawns headless browsers and other subprocesses that don't always play nicely with unprivileged LXC containers.
- A VM gives you a clean, isolated Linux that behaves exactly like a normal server.
- Snapshots of a VM are simpler to roll back if something breaks.

If you're comfortable with LXC, the steps below still apply, just skip the VM creation parts.

---

## Step 1: Pick the right ISO

Use **Ubuntu Server 24.04 LTS** (or 22.04 LTS) for this guide. It's well supported, has Node.js 20+ available without extra repos, and matches the systemd setup the ClaudeClaw setup wizard expects.

1. On your laptop, download the Ubuntu Server ISO from [ubuntu.com/download/server](https://ubuntu.com/download/server).
2. In the Proxmox web UI, click your storage that holds ISOs (usually `local`) → **ISO Images** → **Upload**, and upload the file.

> **Debian 12** also works fine if you prefer it. Anything else (Fedora, Arch, Alpine) will work too, but the package commands in this guide assume `apt`.

---

## Step 2: Create the VM in Proxmox

In the Proxmox web UI, click **Create VM** in the top right and fill out each tab:

| Tab | Setting | Value |
|-----|---------|-------|
| General | Name | `claudeclaw` |
| OS | ISO image | the Ubuntu ISO you just uploaded |
| System | Machine | `q35` (default is fine) |
| System | BIOS | OVMF (UEFI) is fine, or stick with default SeaBIOS |
| System | Qemu Agent | **check this box** (makes shutdown/IP detection cleaner) |
| Disks | Disk size | **20 GB** minimum, 40 GB recommended |
| Disks | SSD emulation | check it if your storage is SSD |
| CPU | Cores | **2** (4 if you plan to run multiple agents) |
| CPU | Type | `host` (best performance) |
| Memory | RAM | **2048 MB** minimum, **4096 MB** recommended |
| Network | Bridge | `vmbr0` (default) |
| Network | Model | VirtIO (default) |

Click **Finish**, then **Start** the VM and open the **Console** tab to install Ubuntu.

---

## Step 3: Install Ubuntu

The installer is mostly click-through. A few choices that matter:

- **Network**: accept the default DHCP. You can give it a static IP later from your router.
- **Storage**: use the entire disk, no LVM unless you have a reason.
- **Profile**: pick a username you'll remember (e.g. `claudeclaw`) and a strong password.
- **SSH**: **check "Install OpenSSH server"**. You want this so you can leave the noisy Proxmox console behind.
- **Snaps**: skip everything. You don't need Docker, microk8s, etc.

When the installer finishes, reboot the VM. **Eject the install ISO first** in Proxmox (Hardware → CD/ROM → Edit → Do not use any media) or it'll boot back into the installer.

---

## Step 4: Connect via SSH

Find the VM's IP address. From the VM console, log in and run:

```bash
ip a
```

Look for the `inet` line under your network interface (usually `ens18`). It'll look like `192.168.1.42`. From your laptop:

```bash
ssh claudeclaw@192.168.1.42
```

From here on, every command in this guide is run from your SSH session.

> **Tip**: while you're here, give the VM a static IP from your router's DHCP reservation page. That way the IP never changes.

---

## Step 5: Update the system and install prerequisites

Get everything current and install the basics:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git build-essential ca-certificates
```

Now install **Node.js 20** using NodeSource (the version in Ubuntu's default repos is too old):

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

Verify the versions:

```bash
node --version    # should print v20.x.x or higher
npm --version
git --version
```

If `node --version` shows v20 or higher, you're good.

---

## Step 6: Install the Claude Code CLI and log in

ClaudeClaw spawns the actual `claude` CLI under the hood, so it has to be installed and authenticated:

```bash
sudo npm install -g @anthropic-ai/claude-code
claude --version
```

Now log in. This is the **trickiest part on a headless server** because `claude login` opens a browser, which you don't have on a Linux VM:

```bash
claude login
```

It prints a URL with a one-time code. **Copy the URL, paste it into the browser on your laptop**, complete the login there, and Claude on the VM will pick it up automatically.

> **Alternative**: if you'd rather use pay-per-token API billing instead of a Claude subscription, skip `claude login` and add `ANTHROPIC_API_KEY=sk-ant-...` to your `.env` file in Step 9. The setup wizard will ask about this.

---

## Step 7: Create your Telegram bot

If you don't already have one:

1. Open Telegram on your phone, search for **@BotFather**.
2. Send `/newbot`, follow the prompts (name, username).
3. Copy the token it gives you. It looks like `1234567890:AAFxxxxxxx`.

Keep that token in a notes app for the next step.

---

## Step 8: Clone ClaudeClaw

Pick a home for the project. The setup wizard expects to live in your home directory:

```bash
cd ~
git clone https://github.com/earlyaidopters/claudeclaw-os.git
cd claudeclaw-os
npm install
```

`npm install` takes a couple of minutes. Grab a coffee.

---

## Step 9: Run the setup wizard

```bash
npm run setup
```

Answer each prompt:

- **Telegram bot token**: paste the token from Step 7.
- **Chat ID**: when the wizard asks, send any message to your bot from Telegram and press Y. The wizard auto-detects your chat ID.
- **Optional features**: pick what you want. Voice input (Groq) and video analysis (Google) are free to enable. Skip the macOS-specific voice output stuff. it needs the macOS `say` command, which doesn't exist on Linux.
- **PIN, kill phrase, idle lock**: configure these. They're worth the 30 seconds to set up.
- **Specialist agents**: skip if you're not sure. You can add them later.
- **Auto-start**: when the wizard asks if it should install a systemd service, **say yes**. This is the whole point of running on a server.

When the wizard finishes, it builds the project and starts the bot. Send any message to your Telegram bot to confirm it's alive.

---

## Step 10: Confirm the systemd service is healthy

ClaudeClaw installs as a **user systemd service** (not a system-wide one), so the commands need `--user`:

```bash
systemctl --user status claudeclaw
```

You should see `active (running)` in green. Useful commands:

```bash
systemctl --user status claudeclaw       # check status
systemctl --user restart claudeclaw      # restart after config changes
systemctl --user stop claudeclaw         # stop
systemctl --user start claudeclaw        # start
journalctl --user -u claudeclaw -f       # tail the logs
```

---

## Step 11: Make the service survive reboots

User systemd services normally only run while you're logged in. To keep ClaudeClaw running after the VM reboots without anyone logging in, enable **lingering** for your user:

```bash
sudo loginctl enable-linger $USER
```

That's it. Reboot the VM (`sudo reboot`) and the bot will come back up on its own. Check after the reboot:

```bash
systemctl --user status claudeclaw
```

---

## Step 12: Open the dashboard (optional)

The dashboard runs on port `3141` by default. From your laptop, open:

```
http://<VM-IP>:3141/?token=YOUR_DASHBOARD_TOKEN&chatId=YOUR_CHAT_ID
```

Get the token and chat ID from `~/claudeclaw-os/.env` on the VM:

```bash
grep -E 'DASHBOARD_TOKEN|ALLOWED_CHAT_ID' ~/claudeclaw-os/.env
```

If your laptop and the VM are on the same LAN, that link will work straight away. If you also want to open the dashboard from your phone while you're out of the house, follow the **Cloudflare Tunnel** section in the main README. The instructions are the same on Linux.

> **Firewall note**: Ubuntu Server doesn't enable a firewall by default, so port 3141 is reachable from your LAN out of the box. If you turned on `ufw`, run `sudo ufw allow 3141/tcp` to open it.

---

## Step 13: Take a Proxmox snapshot

Now that everything works, snapshot the VM so you can roll back if a future change breaks things:

1. In the Proxmox web UI, select the VM.
2. Click **Snapshots** → **Take Snapshot**.
3. Name it something like `clean-install-working`.

If anything ever goes sideways, you can revert to this exact state in seconds.

---

## Updating ClaudeClaw later

When a new version is released:

```bash
cd ~/claudeclaw-os
git pull
npm install
npm run migrate
npm run build
systemctl --user restart claudeclaw
```

---

## Troubleshooting

**Bot doesn't reply on Telegram.**
Check the logs: `journalctl --user -u claudeclaw -f`. The most common causes are a wrong bot token, a wrong chat ID in `.env`, or `claude login` not having been completed on the VM.

**`systemctl --user` says "Failed to connect to bus".**
This happens if you switched users with `sudo su` instead of logging in directly. SSH in fresh as your normal user.

**Service runs but stops when you log out.**
You forgot Step 11. Run `sudo loginctl enable-linger $USER`.

**Claude CLI complains it isn't logged in.**
SSH in, run `claude login` again, then restart the service: `systemctl --user restart claudeclaw`. Auth tokens live in `~/.claude/`, which is per-user, so make sure you logged in as the same user the service runs as.

**Dashboard shows 401 Unauthorized.**
The `token=` value in the URL doesn't match `DASHBOARD_TOKEN` in `.env`. Copy it again carefully.

**VM runs out of disk after a few months.**
SQLite memory plus Telegram media uploads grow over time. Either resize the VM disk in Proxmox (Hardware → Hard Disk → Resize disk, then `sudo growpart` + `sudo resize2fs` inside the VM), or periodically clear `~/claudeclaw-os/workspace/uploads/`.

---

## Linux VM (homelab + Tailscale) vs macmini: what changes

ClaudeClaw runs on either one, but the two environments have different strengths. Read this before you pick so you're not surprised later.

### Things a Linux VM does *better* than a macmini

| Area | Why the VM wins |
|------|-----------------|
| **Always-on reliability** | No display sleep, no "Mac woke up to install updates", no Spotlight hammering the disk. systemd + `enable-linger` is rock solid. |
| **Snapshots and rollback** | Proxmox snapshots take seconds. On a macmini you're stuck with Time Machine or manual backups. |
| **Resource isolation** | The bot can't steal CPU from your day-to-day work, because there is no day-to-day work on it. |
| **Reproducibility** | Rebuild the whole thing from the ISO in 20 minutes. A macmini's config drifts over the years. |
| **Cost** | A VM is free if you already have a Proxmox box. A macmini is $600+. |
| **Remote access via Tailscale** | Install Tailscale on the VM (`curl -fsSL https://tailscale.com/install.sh \| sh && sudo tailscale up`) and the dashboard becomes reachable from any of your devices at `http://claudeclaw:3141/...` with zero port forwarding, zero Cloudflare tunnel, zero public exposure. |
| **MagicDNS + HTTPS** | Turn on MagicDNS and HTTPS certs in the Tailscale admin panel and you can skip the whole `DASHBOARD_URL` + Cloudflared flow entirely. The dashboard just works over the tailnet. |
| **Headless by design** | No keyboard, no monitor, no one logs in and accidentally closes the terminal window. |
| **Multi-agent scaling** | Want five specialist agents? Bump the VM from 2 → 4 cores in Proxmox. On a macmini you're capped at whatever you bought. |

### Things a macmini does that a Linux VM *cannot* (or can only fake)

These are the genuine functional gaps. If any of these matter to you, either keep the macmini or set up a hybrid (bot on Linux, macOS-only skills triggered from a laptop).

| Capability | Status on Linux VM | Workaround |
|------------|--------------------|-----------|
| **macOS `say` TTS fallback** | Not available | Use ElevenLabs or Gradium for voice output. There is no free local fallback on Linux. |
| **Apple Messages / iMessage** | Not available | None. iMessage requires a real Apple device. |
| **Apple Notes, Apple Mail, Apple Calendar, Reminders** | Not available | Use the bundled Gmail and Google Calendar skills, or the Google Workspace CLI. |
| **Apple Shortcuts** | Not available | Write plain shell scripts or Claude skills. |
| **macOS Keychain access** | Not available | Store secrets in `.env` (already how ClaudeClaw does it). |
| **AppleScript / JXA automation** | Not available | None for macOS apps. For cross-platform tasks, use the `agent-browser` skill. |
| **Xcode, iOS builds, iOS simulator** | Not available | None. You need a Mac for any Apple-platform build. |
| **macOS `open` command, Spotlight, Finder tags** | Not available | Use `xdg-open` and regular `find` / `locate`. |
| **Touch ID / Apple Pay prompts** | Not available | None. |
| **launchd agents** | Not applicable | Use the systemd user service installed by the setup wizard (covered in Step 10). |
| **Local ML accelerated by Metal / Apple Neural Engine** | Not available (unless the VM has a GPU passed through) | Use cloud APIs (Groq for Whisper, Gemini for video) or pass a GPU to the VM. |
| **"Continuity" with iPhone/iPad** (AirDrop, Handoff, SMS relay) | Not available | None. |

### The practical shape of a Tailscale-based homelab deployment

If you're going the Linux-in-Proxmox route, here's the setup that pays off the most:

1. **Tailscale on the VM** for SSH and dashboard access from anywhere. Set a hostname like `claudeclaw` in Tailscale → MagicDNS gives you `http://claudeclaw:3141/...` from any device in your tailnet.
2. **Tailscale on your phone** so `/dashboard` links open natively over the VPN with no Cloudflare tunnel.
3. **Tailscale SSH** (`sudo tailscale up --ssh`) so you stop managing SSH keys and use Tailscale ACLs instead.
4. **Tailscale ACLs** if you want to share the dashboard with a partner or teammate, scoped to their device only.
5. **Proxmox backups** to a NAS or external drive on top of snapshots, so a dead Proxmox host isn't a data-loss event.

### When to keep the macmini

Keep the macmini if **any** of these are true for you:

- You rely on iMessage, Apple Notes, Apple Mail, or Reminders as part of your daily flow and want ClaudeClaw to read/write them.
- You're building iOS or macOS apps and want Claude to run `xcodebuild`.
- You use the macOS `say` local TTS fallback and don't want to pay ElevenLabs or register for Gradium.
- Your workflow depends on Apple Shortcuts.

Otherwise, a Linux VM on Proxmox with Tailscale is the stronger long-term home for a 24/7 assistant.

---

## Hardening the VM (on top of Tailscale)

Tailscale already takes you out of the public internet, which is 80% of the battle. These steps tighten up the remaining 20%. Do them in order.

### 1. Lock the firewall to the tailnet

Even on a home LAN, the VM shouldn't be answering random probes from IoT devices, roommates, or a compromised machine on the same subnet. Drop everything that isn't Tailscale:

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow traffic from the Tailscale interface only
sudo ufw allow in on tailscale0

# Keep a LAN fallback for SSH in case Tailscale breaks
# (remove this line later once you're confident)
sudo ufw allow from 192.168.0.0/16 to any port 22 proto tcp

sudo ufw enable
sudo ufw status verbose
```

Now the dashboard on port 3141 is only reachable over the tailnet, even though the app still binds to `0.0.0.0` internally. If you want belt-and-braces, you can also set `DASHBOARD_HOST` if your version exposes it, but the firewall rule above is enough.

### 2. SSH: keys only, no passwords, no root

Generate an SSH key on your laptop (if you don't already have one), copy it up, then lock down sshd:

```bash
# On your laptop
ssh-copy-id claudeclaw@<VM-IP>

# On the VM
sudo nano /etc/ssh/sshd_config.d/99-hardening.conf
```

Paste:

```
PasswordAuthentication no
PermitRootLogin no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

Save, then:

```bash
sudo systemctl restart ssh
```

**Better still: turn on Tailscale SSH and disable the normal sshd entirely.** That way even a stolen key doesn't get anyone in unless they're on your tailnet:

```bash
sudo tailscale up --ssh
sudo systemctl disable --now ssh
```

You now SSH in with `ssh claudeclaw@claudeclaw` (Tailscale MagicDNS) and auth happens through your Tailscale identity.

### 3. Automatic security updates

Unattended upgrades patch CVEs without you thinking about it:

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Answer "Yes" when it asks. Then verify it's running:

```bash
sudo systemctl status unattended-upgrades
```

Reboots after kernel updates are still manual — schedule a monthly `sudo reboot` or install `needrestart` and let it tell you when one's needed.

### 4. Lock down `.env` (it's full of secrets)

The `.env` file has your Telegram token, `DASHBOARD_TOKEN`, `DB_ENCRYPTION_KEY`, and every API key you added. Anyone who reads it owns your bot:

```bash
chmod 600 ~/claudeclaw-os/.env
ls -la ~/claudeclaw-os/.env    # should show -rw------- claudeclaw claudeclaw
```

While you're at it, make sure the SQLite DB isn't world-readable either:

```bash
chmod 700 ~/claudeclaw-os/store
chmod 600 ~/claudeclaw-os/store/claudeclaw.db
```

### 5. Harden the systemd service

Edit the unit the setup wizard dropped at `~/.config/systemd/user/claudeclaw.service` and add these lines inside the `[Service]` block, above `[Install]`:

```ini
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=%h/claudeclaw-os %h/.claude %h/.config/claudeclaw
PrivateTmp=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictSUIDSGID=true
LockPersonality=true
```

Then reload and restart:

```bash
systemctl --user daemon-reload
systemctl --user restart claudeclaw
systemctl --user status claudeclaw
```

If the bot fails to start, it usually means one of your skills writes to a path outside `ReadWritePaths` — add the offending path and retry. Don't just remove the hardening.

### 6. Turn on ClaudeClaw's own security features

This is the layer that protects you if an attacker gets as far as Telegram itself. In your `.env` (the setup wizard prompts for these, but double check):

```
SECURITY_PIN=1234                  # required before the bot will act
SECURITY_IDLE_LOCK_MINUTES=15      # auto-lock after inactivity
SECURITY_KILL_PHRASE=howlongdoyougot   # one-shot panic nuke
```

Send `/status` in Telegram to verify PIN and idle-lock are active. Rotate `DASHBOARD_TOKEN` and `SECURITY_PIN` every few months by regenerating and restarting the service.

### 7. Back up the SQLite database

Snapshots protect you from "I broke the VM." They don't help if `claudeclaw.db` corrupts or a skill deletes something. Add a nightly dump to a different disk (or to your NAS over Tailscale):

```bash
mkdir -p ~/backups
cat > ~/backup-claudeclaw.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
STAMP=$(date +%Y%m%d-%H%M)
DEST="$HOME/backups/claudeclaw-$STAMP.db"
sqlite3 "$HOME/claudeclaw-os/store/claudeclaw.db" ".backup '$DEST'"
gzip -f "$DEST"
# Keep 14 days
find "$HOME/backups" -name 'claudeclaw-*.db.gz' -mtime +14 -delete
EOF
chmod +x ~/backup-claudeclaw.sh

# Run it nightly at 03:30
(crontab -l 2>/dev/null; echo "30 3 * * * $HOME/backup-claudeclaw.sh") | crontab -
```

`sqlite3 .backup` is safe to run while the bot is live. A plain `cp` of the DB file isn't.

### 8. Keep Node and npm dependencies patched

Every few weeks:

```bash
cd ~/claudeclaw-os
npm audit
npm outdated
```

Actually act on the output. `npm audit fix` usually works; anything that requires a major version bump, read the changelog first.

### 9. Don't run as root, don't `sudo npm install -g` your bot

You're already fine here if you followed the guide — ClaudeClaw runs as your regular user via a user systemd unit. If you ever feel tempted to "just use root to make the permissions work", stop and fix the permissions instead.

### 10. Monitor what the bot is doing

`journalctl --user -u claudeclaw -f` is your friend. Set up a log alert (even just a cron that greps for `error` and pushes a Telegram message via the notify script) so you notice crash loops, rate limits, or API key failures within minutes instead of days.

---

### What you should *not* bother with on a homelab VM

Skip these unless you have a specific reason, they're overkill for a personal bot:

- SELinux / AppArmor custom profiles (the systemd hardening above is enough)
- Fail2ban (you disabled password SSH and firewalled to tailnet, there's nothing to ban)
- A reverse proxy (nginx/Caddy) in front of the dashboard (Tailscale HTTPS handles this)
- Running the bot inside Docker on top of the VM (adds complexity with no meaningful isolation gain over the systemd unit)
- Full-disk encryption on a homelab VM whose disk never leaves your house (useful on a laptop, paranoid here)
