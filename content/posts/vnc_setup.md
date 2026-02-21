+++
title = 'Just Another VNC TLS+Passwd Auth and SSH Tunnel Setup'
date = '2026-02-20T15:08:17-06:00'
draft = false
tags = ["ssh", "ubuntu", "homelab", "vnc"]
categories = ["Homelab", "DevOps"]
description = "A guide to setting up VNC Server with TLS+Passwd Auth and connecting via SSH Tunnel on Ubuntu 24.04."
layout = "post"
+++

# Ubuntu 24.04 Cinnamon VNC Setup (Cinnamon) with SSH Tunnel

## Introduction

I know this is yet another VNC blog post among many out there, but I
wanted to document the exact steps I took to set up my local
infrastructure for future reference. Writing things down helps me keep
track of what I'm doing and how I did it especially when I revisit
something months later.

Below are the steps I used to configure two Ubuntu 24.04 systems (server
& client) running Cinnamon for secure SSH-tunneled VNC connectivity with
TLS and Password authentication.

------------------------------------------------------------------------

## Why This Setup?

My goal was to support two use cases:

-   A **virtual desktop session** that runs independently of the
    physical display (useful for headless or persistent sessions)
-   A **shared display session** that connects to the live desktop

TigerVNC supports both models cleanly, and when paired with SSH
tunneling, it keeps everything private without exposing VNC ports
directly to the network.

------------------------------------------------------------------------

## Overview

Two VNC configurations are set up on remote server `media-01`:

-   **Virtual desktop** (`:2`, port `5902`) --- isolated session via
    `tigervncserver`
-   **Shared display** (`:1`, port `5901`) --- live desktop session via
    `x0vncserver`

Both are bound to `localhost` only and accessed via SSH tunnel.

### Architecture (Simplified)

    Client
      |
    SSH Tunnel (5901 / 5902)
      |
    TLS (X509Vnc + Password)
      |
    media-01
      ├─ x0vncserver (:1)    -> shared live display
      └─ tigervncserver (:2) -> virtual desktop

------------------------------------------------------------------------

## Server Prerequisites

### Static IP Configuration

Before setting up VNC, I configured a static IP for `media-01` to ensure
it is always reachable at `192.168.2.124`. This was done via my router.

![Static IP configuration](/img/vnc_setup/static-ip.png)

------------------------------------------------------------------------

## Packages

``` bash
sudo apt install tigervnc-standalone-server
sudo apt install tigervnc-scraping-server
sudo apt install dbus-x11
```

------------------------------------------------------------------------

## Password Setup

``` bash
vncpasswd ~/.vnc/passwd
chmod 600 ~/.vnc/passwd
```

> **Note:** VNC passwords are limited to 8 characters and use weak DES
> obfuscation. File permissions plus SSH tunneling are the real security
> controls here.

------------------------------------------------------------------------

## Desktop Session Startup Script

Create `~/.vnc/xstartup`:

``` bash
#!/bin/bash
exec /usr/bin/cinnamon-session
```

Make it executable:

``` bash
chmod +x ~/.vnc/xstartup
```

------------------------------------------------------------------------

## Determining the Correct DISPLAY

Do not assume the physical display is `:1`. On many Ubuntu systems it is
`:0`.

Verify using:

``` bash
echo $DISPLAY
loginctl list-sessions
```

Adjust your systemd service accordingly.

------------------------------------------------------------------------

## Systemd Service --- Virtual Desktop (`:2`)

I created a systemd file at `/etc/systemd/system/tigervncserver@.service` so that the virtual VNC server
is always online on startup of Ubuntu:

``` ini
[Unit]
Description=TigerVNC Server
After=network.target

[Service]
Type=forking
User=box02
WorkingDirectory=/home/box02

PIDFile=/home/box02/.vnc/%H:%i.pid
ExecStartPre=-/usr/bin/vncserver -kill :%i
ExecStart=/usr/bin/vncserver :%i -localhost -SecurityTypes X509Vnc -X509Key /home/box02/.vnc/x509_key.pem -X509Cert /home/box02/.vnc/x509_cert.pem -PasswordFile /home/box02/.vnc/passwd
ExecStop=/usr/bin/vncserver -kill :%i

Restart=on-failure
RestartSec=5

# Optional hardening
ProtectSystem=full
ProtectHome=true
PrivateTmp=true
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

Enable and start:

``` bash
sudo systemctl daemon-reload
sudo systemctl enable tigervncserver@2.service
sudo systemctl start tigervncserver@2.service
```

------------------------------------------------------------------------

## Systemd Service --- Shared Display (`:1`)

I created a systemd file at `/etc/systemd/system/x0vncserver.service` so that the shared VNC server
is always online on startup of Ubuntu:

``` ini
[Unit]
Description=TigerVNC Shared Display Server
After=network.target

[Service]
Type=simple
User=box02
Environment=DISPLAY=:1
Environment=XAUTHORITY=/home/box02/.Xauthority

ExecStart=/usr/bin/x0vncserver -display :1 -localhost -fg -SecurityTypes X509Vnc -X509Key /home/box02/.vnc/x509_key.pem -X509Cert /home/box02/.vnc/x509_cert.pem -PasswordFile /home/box02/.vnc/passwd

Restart=on-failure
RestartSec=5

ProtectSystem=full
ProtectHome=true
PrivateTmp=true
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

> Adjust `DISPLAY` if your system uses `:0`.

------------------------------------------------------------------------

## Useful Commands

Here are some commands I used on the remote server to restart and view service logs.

``` bash
journalctl -fu tigervncserver@2.service
journalctl -fu x0vncserver.service

sudo systemctl restart tigervncserver@2.service
sudo systemctl restart x0vncserver.service

vncserver -list
```

------------------------------------------------------------------------

## Client Setup

Install VNC viewer packages:

``` bash
sudo apt install tigervnc-viewer tigervnc-tools
```

### SSH Key for Passwordless Tunnel

``` bash
ssh-keygen -t ed25519 -C "vnc-tunnel" -f ~/.ssh/media_01_id_ed25519
ssh-copy-id -i ~/.ssh/media_01_id_ed25519.pub box02@192.168.2.124
```

------------------------------------------------------------------------

## Wrapper Connect Script

At this point I wanted a one-shot script to bring up the VNC session without
having to remember everything involved, saved to `~/bin/vnc-connect`:

``` bash
#!/bin/bash

set -e

export SSH_SERVER=192.168.2.124
export REMOTE_PORT=5901
export LOCAL_PORT=5901
export IDENTITY_FILE="${HOME}/.ssh/media_01_id_ed25519"
export CONFIG="${HOME}/.config/tigervnc/media-01-shared.tigervnc"
export PASSWD="${HOME}/.vnc/passwd"

SSH_PID=$(pgrep -f "ssh -f -N -i ${IDENTITY_FILE} -L ${LOCAL_PORT}" || true)
if [ -n "$SSH_PID" ]; then
  echo "Lingering ssh tunnel found killing it now"
  kill $SSH_PID
else
  echo "No ssh tunnel found starting one now"
fi

ssh -f -N -i ${IDENTITY_FILE} -L ${LOCAL_PORT}:127.0.0.1:${REMOTE_PORT} box02@${SSH_SERVER}
sleep 1
nohup xtigervncviewer -passwd "${PASSWD}" "${CONFIG}" 2>&1 > /tmp/vncviewer.log &
```

Make it executable:

``` bash
chmod +x ~/bin/vnc-connect
```

Ensure `~/bin` is in PATH:

``` bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

------------------------------------------------------------------------

## Example Invocation

Below is an example of me invoking the one-shot script `vnc-connect` with the TigerVNC viewer
session active.

![vnc-connect invocation with TigerVNC viewer](/img/vnc_setup/vnc-connect.png)

------------------------------------------------------------------------

## Manual SSH Tunnel (Optional)

You can certainly tunnel manually and below is the command if you want to:

``` bash
ssh -L 5901:127.0.0.1:5901 box02@192.168.2.124 -N
```

Then connect your VNC viewer to `127.0.0.1:5901`.

------------------------------------------------------------------------

## Addendum — TLS with X509Vnc

After the initial setup I decided to add TLS encryption to both VNC servers using
TigerVNC's built-in `X509Vnc` security type. Since I was already using SSH tunneling,
this is more of a defense-in-depth measure than a strict necessity — but it's good practice.

### Generating the Certificate

A self-signed certificate was generated on `media-01` using a small script saved to `~/gen-certs.sh`:

``` bash
#!/usr/bin/env bash

set -e

mkdir -p ~/.vnc

openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:P-384 \
  -keyout ~/.vnc/x509_key.pem \
  -out ~/.vnc/x509_cert.pem \
  -days 3650 -nodes \
  -subj "/CN=media-01" \
  -addext "subjectAltName=IP:192.168.2.124,IP:127.0.0.1,DNS:localhost" \
  -addext "extendedKeyUsage=serverAuth" \
  -addext "keyUsage=critical,digitalSignature,keyEncipherment" \
  -addext "basicConstraints=critical,CA:FALSE"

chmod 600 ~/.vnc/x509_key.pem
chmod 644 ~/.vnc/x509_cert.pem

echo "Certificate generated successfully:"
openssl x509 -in ~/.vnc/x509_cert.pem -text -noout
```

``` bash
chmod +x ~/gen-certs.sh
./gen-certs.sh
```

### Client Certificate

Copy the server certificate to the client machine so the viewer can verify the server's identity:

``` bash
scp box02@192.168.2.124:~/.vnc/x509_cert.pem ~/.vnc/x509_ca.pem
```

### TigerVNC Client Config

At this point I just updated `~/.config/tigervnc/media-01-shared.tigervnc` with the below contents 
to get the client working with TLS auth. It is a direct copy in its entirety: 

``` ini
TigerVNC Configuration file Version 1.0

ServerName=127.0.0.1:5901
X509CA=/home/boringpc/.vnc/x509_ca.pem
SecurityTypes=X509Vnc
ReconnectOnError=1
Shared=0
AutoSelect=0
FullColor=1
LowColorLevel=2
PreferredEncoding=ZRLE
CustomCompressLevel=0
CompressLevel=6
NoJPEG=1
QualityLevel=8
FullScreen=1
FullScreenMode=Current
FullScreenSelectedMonitors=1
ViewOnly=0
EmulateMiddleButton=0
DotWhenNoCursor=0
AcceptClipboard=1
SendClipboard=1
SendPrimary=1
SetPrimary=1
MenuKey=F8
FullscreenSystemKeys=1
desktopSize=1920x1080
```

> **Note:** `ServerName` uses `127.0.0.1` instead of `localhost` to match the certificate SANs
> and avoid a hostname mismatch warning.

------------------------------------------------------------------------

## Security Considerations

-   Bind VNC to `localhost` only
-   Access via SSH tunnel
-   Avoid exposing ports 5901/5902
-   Consider firewall rules even on LAN
-   Use key-based SSH authentication
-   Use TLS (`X509Vnc`) for defense in depth

------------------------------------------------------------------------

## Troubleshooting

-   **Black screen:** Ensure `dbus-x11` is installed.
-   **Session exits immediately:** Check `~/.vnc/xstartup` permissions.
-   **Permission denied:** Verify `chmod 600 ~/.vnc/passwd`.
-   **Wrong display:** Confirm with `echo $DISPLAY`.
-   **Wayland issues:** Ensure you are running an X11 session if required.
-   **Certificate hostname mismatch:** Use `127.0.0.1` instead of `localhost` in the TigerVNC config.

------------------------------------------------------------------------

## Closing Thoughts

This setup has worked reliably for me and gives me both a persistent
virtual session and live desktop control, all without exposing VNC
directly to the network. Documenting it here ensures I can quickly
reproduce it later.

> I used Claude and ChatGPT as sounding boards while working through this
> setup. They were helpful for troubleshooting and reviewing the post, but
> the ideas, decisions, and final edits are mine.