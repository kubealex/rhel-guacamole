# RHEL 10 GNOME Remote Login via RDP

This guide configures a RHEL 10 workstation for GNOME Remote Desktop **Remote Login**.

This mode allows an RDP client such as Apache Guacamole to reach the GDM login screen even when no user is already logged into GNOME.

Architecture:

```text
RDP Client
    |
    | TCP/3389
    v
GNOME Remote Desktop
(system daemon)
    |
    | RDP gateway authentication
    v
GDM
    |
    | Linux authentication
    v
GNOME Wayland session
    |
    +-- PipeWire
    |
    v
Remote Desktop
```

## 1. Install the required components

```bash
sudo dnf install -y \
  gdm \
  gnome-shell \
  gnome-session \
  gnome-remote-desktop \
  freerdp \
  pipewire \
  pipewire-pulseaudio \
  wireplumber
```

Enable graphical boot:

```bash
sudo systemctl set-default graphical.target
```

Enable GDM:

```bash
sudo systemctl enable --now gdm.service
```

Verify:

```bash
systemctl is-active gdm.service
```

Expected:

```text
active
```

## 2. Create the TLS certificate

```bash
sudo mkdir -p /etc/gnome-remote-desktop

sudo openssl req \
  -x509 \
  -newkey rsa:3072 \
  -nodes \
  -days 825 \
  -subj "/CN=$(hostname -f)" \
  -keyout /etc/gnome-remote-desktop/rdp-tls.key \
  -out /etc/gnome-remote-desktop/rdp-tls.crt
```

The system GNOME Remote Desktop daemon runs as:

```text
gnome-remote-desktop
```

Give it access to the certificate and private key:

```bash
sudo chown \
  gnome-remote-desktop:gnome-remote-desktop \
  /etc/gnome-remote-desktop/rdp-tls.crt \
  /etc/gnome-remote-desktop/rdp-tls.key

sudo chmod 644 /etc/gnome-remote-desktop/rdp-tls.crt
sudo chmod 600 /etc/gnome-remote-desktop/rdp-tls.key

sudo restorecon -RFv /etc/gnome-remote-desktop
```

Verify access:

```bash
sudo runuser -u gnome-remote-desktop -- \
  openssl x509 \
  -in /etc/gnome-remote-desktop/rdp-tls.crt \
  -noout -subject -dates

sudo runuser -u gnome-remote-desktop -- \
  openssl pkey \
  -in /etc/gnome-remote-desktop/rdp-tls.key \
  -noout -check
```

The private-key test should report:

```text
Key is valid
```

## 3. Configure Remote Login

Configure TLS:

```bash
sudo grdctl --system rdp set-tls-cert \
  /etc/gnome-remote-desktop/rdp-tls.crt

sudo grdctl --system rdp set-tls-key \
  /etc/gnome-remote-desktop/rdp-tls.key
```

Configure the RDP gateway credentials:

```bash
sudo grdctl --system rdp set-credentials \
  rdpuser 'CHANGE_ME_RDP_PASSWORD'
```

`rdpuser` is not required to exist in `/etc/passwd`.

It is an RDP gateway credential managed by GNOME Remote Desktop and is separate from the Linux account used at GDM.

Configure the RDP service:

```bash
sudo grdctl --system rdp disable-view-only
sudo grdctl --system rdp disable-port-negotiation
sudo grdctl --system rdp enable
```

Enable the daemon:

```bash
sudo systemctl enable --now gnome-remote-desktop.service
sudo systemctl restart gnome-remote-desktop.service
```

## 4. Configure PipeWire for the Linux user

GNOME Remote Desktop uses PipeWire for the desktop stream.

Log in as the Linux user that will use the remote desktop, for example `sysadmin`, and run:

```bash
systemctl --user enable --now \
  pipewire.socket \
  pipewire-pulse.socket \
  wireplumber.service
```

Verify:

```bash
systemctl --user status \
  pipewire.socket \
  pipewire.service \
  wireplumber.service
```

Check processes:

```bash
ps -ef | grep -E '[p]ipewire|[w]ireplumber'
```

The user's runtime directory should contain:

```bash
ls -l "$XDG_RUNTIME_DIR/pipewire-0"
```

### Configuring sysadmin from root

If the workstation is being provisioned remotely and you are already root:

```bash
UID_SYSADMIN=$(id -u sysadmin)
RUNTIME="/run/user/$UID_SYSADMIN"

runuser -u sysadmin -- env \
  XDG_RUNTIME_DIR="$RUNTIME" \
  DBUS_SESSION_BUS_ADDRESS="unix:path=$RUNTIME/bus" \
  systemctl --user enable --now \
    pipewire.socket \
    pipewire-pulse.socket \
    wireplumber.service
```

## 5. Verify RDP

Check GRD:

```bash
sudo grdctl --system status
```

Expected:

```text
Overall:
    Unit status: active

RDP:
    Status: enabled
    Port: 3389
    TLS certificate: /etc/gnome-remote-desktop/rdp-tls.crt
    TLS key: /etc/gnome-remote-desktop/rdp-tls.key
    Username: rdpuser
```

Check the listener:

```bash
sudo ss -lntp | grep ':3389'
```

Expected:

```text
LISTEN ... *:3389 ... gnome-remote-de...
```

Check logs:

```bash
sudo journalctl \
  -u gnome-remote-desktop.service \
  -n 50 \
  --no-pager
```

A successful startup contains:

```text
RDP server started
```

The following is harmless on a VM without a TPM:

```text
Init TPM credentials failed because No TPM device found,
using GKeyFile as fallback.
```

## 6. Login

Connect the RDP client using:

```text
Host:       <RHEL_WORKSTATION>
Port:       3389
Username:   rdpuser
Password:   <RDP_GATEWAY_PASSWORD>
```

GNOME Remote Desktop then presents GDM.

Authenticate there using the actual Linux account:

```text
Username: sysadmin
Password: <LINUX_PASSWORD>
```

The complete authentication flow is therefore:

```text
RDP connection
      |
      | rdpuser
      v
GNOME Remote Desktop
      |
      v
GDM
      |
      | sysadmin
      v
GNOME user session
```

## Troubleshooting

| Symptom | Check | Cause / Fix |
|---|---|---|
| `3389` is not listening | `grdctl --system status` | Ensure RDP is enabled |
| `RDP TLS certificate and key not yet configured properly` | `ls -l /etc/gnome-remote-desktop/rdp-tls.*` | GRD daemon cannot read the certificate/key |
| TLS key is `root:root 0600` | `systemctl show gnome-remote-desktop -p User` | Change ownership to `gnome-remote-desktop:gnome-remote-desktop` |
| GRD service active but no RDP listener | `journalctl -u gnome-remote-desktop` | Usually incomplete TLS configuration |
| GDM does not appear | `systemctl status gdm` | Install/enable GDM and graphical target |
| Login succeeds but RDP disconnects immediately | Search journal for `Couldn't connect pipewire context` | User PipeWire session is not running |
| `Couldn't connect pipewire context` | `systemctl --user status pipewire.socket wireplumber` | Enable/start PipeWire socket and WirePlumber |
| No `/run/user/<UID>/pipewire-0` | Check PipeWire user services | PipeWire has not started for that user |
| `Init TPM credentials failed... using GKeyFile as fallback` | None required | Informational on systems without a TPM |
| Need complete diagnostics | `journalctl -b \| grep -E 'gnome-remote\|pipewire\|wireplumber'` | Correlate GRD handover with user-session startup |
