+++
date = '2026-02-20T15:08:17-06:00'
draft = true
title = 'Just Another VNC and SSH Tunnel Setup'
+++

## Introduction

I know this is yet another VNC blog post among many out there in the wild, however I wanted to document the steps I took to set up my local infrastructure for future reference. It helps me keep track of what I'm doing and how I did it. Below are the series of steps to configure two computers (server & client) running Ubuntu 24.04 with Cinnamon desktop for SSH tunnel and VNC connectivity.

---

## Overview

Two VNC configurations are set up on `media-01`:

- **Virtual desktop** (`:2`, port `5902`) — isolated session via `tigervncserver`
- **Shared display** (`:1`, port `5901`) — live desktop session via `x0vncserver`

Both are bound to localhost only and accessible via SSH tunnel.

---

## Server Prerequisites

### Static IP Configuration

Before setting up VNC, a static IP was configured on `media-01` to ensure the server is always reachable at the same address (`192.168.2.124`). This was done through my Ubiquiti UCG Fiber router, and below is a screenshot of the configuration.

![Static IP configuration](/img/vnc_setup/static-ip.png)

---

## Packages

```bash
# VNC server (virtual desktop)
sudo apt install tigervnc-standalone-server

# VNC server (shared/scraping display)
sudo apt install tigervnc-scraping-server

# Required for desktop session support
sudo apt install dbus-x11
```

---

## Password Setup

```bash
vncpasswd ~/.vnc/passwd
chmod 600 ~/.vnc/passwd
```

> **Note:** VNC passwords are capped at 8 characters (protocol limitation). The passwd file uses weak DES obfuscation — file permissions are the primary protection.

---

## Desktop Session Startup Script

Create `~/.vnc/xstartup` to launch Cinnamon for the virtual desktop:

```bash
vim ~/.vnc/xstartup
```

Add the following contents:

```bash
#!/bin/bash
exec /usr/bin/cinnamon-session
```

```bash
chmod +x ~/.vnc/xstartup
```

Verify:

```bash
cat ~/.vnc/xstartup
```

Expected output:

```
#!/bin/bash
exec /usr/bin/cinnamon-session
```

---

## Systemd Service — Virtual Desktop (`:2`)

Create `/etc/systemd/system/tigervncserver@.service`:

```ini
[Unit]
Description=TigerVNC Server
After=syslog.target network.target

[Service]
Type=forking
User=box02
WorkingDirectory=/home/box02

PIDFile=/home/box02/.vnc/%H:%i.pid
ExecStartPre=-/usr/bin/vncserver -kill :%i
ExecStart=/usr/bin/vncserver :%i -localhost
ExecStop=/usr/bin/vncserver -kill :%i

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable tigervncserver@2.service
sudo systemctl start tigervncserver@2.service
sudo systemctl status tigervncserver@2.service
```

---

## Systemd Service — Shared Display (`:1`)

Create `/etc/systemd/system/x0vncserver.service`:

```ini
[Unit]
Description=TigerVNC Shared Display Server
After=syslog.target network.target

[Service]
Type=simple
User=box02
Environment=DISPLAY=:1
Environment=XAUTHORITY=/home/box02/.Xauthority
ExecStart=/usr/bin/x0vncserver -display :1 -localhost -fg -PasswordFile /home/box02/.vnc/passwd
StandardOutput=journal
StandardError=journal

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable x0vncserver.service
sudo systemctl start x0vncserver.service
sudo systemctl status x0vncserver.service
```

> **Note:** `DISPLAY=:1` is the physical display. Verify with `echo $DISPLAY` if unsure.

---

## Useful Commands

```bash
# View logs
journalctl -fu tigervncserver@2.service
journalctl -fu x0vncserver.service

# Restart services
sudo systemctl restart tigervncserver@2.service
sudo systemctl restart x0vncserver.service

# Check running VNC servers
vncserver -list
```

---

## Client Setup

### Packages

```bash
sudo apt install tigervnc-viewer tigervnc-tools
```

### SSH Key for Passwordless Tunnel

```bash
ssh-keygen -t ed25519 -C "vnc-tunnel" -f ~/.ssh/media_01_id_ed25519
ssh-copy-id -i ~/.ssh/media_01_id_ed25519.pub box02@192.168.2.124
```

### VNC Password File

```bash
mkdir -p ~/.vnc
vncpasswd ~/.vnc/passwd
chmod 600 ~/.vnc/passwd
```

### TigerVNC Connection Config Files

Saved to `~/.config/tigervnc/`. Example for shared display (`media-01-shared.tigervnc`):

```ini
TigerVNC Configuration file Version 1.0

ServerName=localhost:5901
SecurityTypes=VncAuth
PasswordFile=/home/youruser/.vnc/passwd
```

> **Note:** `PasswordFile` in the config file is intentionally not supported by TigerVNC for security reasons. Pass it via `-passwd` on the command line instead.

---

## Connect Script

Saved to `~/bin/vnc-connect`:

```bash
#!/bin/bash

set -e

export SSH_SERVER=192.168.2.124
export REMOTE_PORT=5901
export LOCAL_PORT=5901
export IDENTITY_FILE="${HOME}/.ssh/media_01_id_ed25519"
export CONFIG="${HOME}/.config/tigervnc/media-01-shared.tigervnc"
export PASSWD="${HOME}/.vnc/passwd"

SSH_PID=$(pgrep -f "ssh.*${LOCAL_PORT}:localhost:${REMOTE_PORT}" || true)
if [ -n "$SSH_PID" ]; then
  echo "Lingering ssh tunnel found killing it now"
  kill $SSH_PID
else
  echo "No ssh tunnel found starting one now"
fi

ssh -f -N -i ${IDENTITY_FILE} -L ${LOCAL_PORT}:localhost:${REMOTE_PORT} box02@${SSH_SERVER}
sleep 1
nohup xtigervncviewer -passwd "$PASSWD" "$CONFIG" 2>&1 > /tmp/vncviewer.log &
```

```bash
chmod +x ~/bin/vnc-connect
```

Make sure `~/bin` is in your PATH:

```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

Below is an example of invoking `vnc-connect` with the TigerVNC viewer session active.

![vnc-connect invocation with TigerVNC viewer](/img/vnc_setup/vnc-connect.png)

### SSH Tunnel (Manual)

```bash
# Virtual desktop
ssh -L 5902:localhost:5902 box02@192.168.2.124 -N

# Shared display
ssh -L 5901:localhost:5901 box02@192.168.2.124 -N
```

Then connect VNC viewer to `localhost:5902` or `localhost:5901`.

### Invoke VNC Viewer

```bash
# With config file and password
xtigervncviewer -passwd ~/.vnc/passwd ~/.config/tigervnc/media-01-shared.tigervnc

# Or use the connect script
vnc-connect
```

**P.S.** I used ChatGPT and Claude as sounding boards while reviewing this post. Their feedback helped tighten things up, but the ideas and final edits are mine.
