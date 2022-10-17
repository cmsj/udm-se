# Tailscale on a Unifi UDM SE

## Introduction

There are few comprehensive guides out there for getting Tailscale working on a UDM SE. Plenty exist for UDM Pro, but it seems like the 2.2.x firmware on UDM SE is quite different to UDM Pro's current firmware.

So, this is how I did it.

## Step 1: udm-boot

The convention amongst UDM Pro modifiers seems to be to use a systemd service named `udm-boot` which actually just executes whatever shell scripts are in a particular directory on the UDM Pro's persistent data partition.

The UDM SE has systemd and a persistent data partition, but it's mounted in a different location (`/data` instead of `/mnt/data`), so we will follow the pattern and just adapt the paths.

SSH into your UDM SE and create `/etc/systemd/system/udm-boot.service` with the following contents:

```
[Unit]
Description=Run On Startup UDM
Wants=network-online.target
After=network-online.target

[Service]
Type=forking
ExecStart=bash -c 'mkdir -p /data/on_boot.d && find -L /data/on_boot.d -mindepth 1 -maxdepth 1 -type f -print0 | sort -z | xargs -0 -r -n 1 -- bash -c \'if test -x "$0"; then echo "%n: running $0"; "$0"; else case "$0" in *.sh) echo "%n: sourcing $0"; . "$0";; *) echo "%n: ignoring $0";; esac; fi\''

[Install]
WantedBy=multi-user.target
```

Tell systemd about that service by running:
```
systemctl daemon-reload
systemctl enable /etc/systemd/system/udm-boot.service
systemctl start udm-boot.service
```

Check /data/on_boot.d/ exists

## Step 2: Installing tailscale integration

Fetch the tailscale/udm integration scripts and extract them nto the persistent data partition:
```
wget https://github.com/SierraSoftworks/tailscale-udm/releases/download/v2.2.1/tailscale-udm.tgz
cd /data/
tar xvf ~root/tailscale-udm.tgz
```

## Step 3: Making tailscale work

This is basically a process of updating all of the scripts involved, to refer to `/data` instead of `/mnt/data`:

 * Edit `/data/tailscale/manage.sh` and change all `/mnt/data` to `/data` (should be about 3 edits)
 * Run `/data/tailscale/manage.sh install` (this actually
 * Edit `/data/on_boot.d/10-tailscaled.sh` and change all `/mnt/data` to `/data` (should only be one edit)
 * Run `/data/tailscale/manage.sh on-boot` - this checks if a tailscale update exists and then starts the service
 * You should see that tailscale starts and prints out a `login.tailscale.com` URL. Open the URL and authorize the device
 * Check everything is good with `/data/tailscale/tailscale status`

## Step 4: Celebrate

You are done.

