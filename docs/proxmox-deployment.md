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
