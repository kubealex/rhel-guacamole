# RHEL SSH Server Configuration

This guide configures a RHEL server for SSH access via Apache Guacamole.

Architecture:

```text
Guacamole (guacd)
    |
    | TCP/22
    v
OpenSSH Server (sshd)
    |
    | Linux authentication
    v
Shell session
```

## 1. Install OpenSSH Server

OpenSSH server is typically pre-installed on RHEL. Verify:

```bash
rpm -q openssh-server
```

If not installed:

```bash
sudo dnf install -y openssh-server
```

## 2. Enable and start sshd

```bash
sudo systemctl enable --now sshd.service
```

Verify:

```bash
systemctl is-active sshd.service
```

Expected:

```text
active
```

## 3. Configure the firewall

```bash
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

Verify:

```bash
sudo firewall-cmd --list-services
```

`ssh` should appear in the output.

## 4. Verify SSH access

Check the listener:

```bash
sudo ss -lntp | grep ':22'
```

Expected:

```text
LISTEN ... *:22 ... sshd
```

Test from the Guacamole server:

```bash
ssh <username>@<RHEL_SERVER_IP>
```

## 5. Optional: SSH hardening

Edit `/etc/ssh/sshd_config`:

```bash
sudo vi /etc/ssh/sshd_config
```

Recommended settings:

```text
PermitRootLogin no
PasswordAuthentication yes
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

Restart sshd after changes:

```bash
sudo systemctl restart sshd.service
```

## 6. Key-based authentication (optional)

Generate a key pair:

```bash
ssh-keygen -t ed25519 -C "guacamole-access"
```

Copy the public key to the target server:

```bash
ssh-copy-id <username>@<RHEL_SERVER_IP>
```

The private key can then be pasted into the Guacamole SSH connection configuration under **SFTP / Authentication**.

## Troubleshooting

| Symptom | Check | Cause / Fix |
|---|---|---|
| Connection refused | `ss -lntp \| grep :22` | sshd is not running |
| Connection timeout | `firewall-cmd --list-services` | SSH not allowed through firewall |
| Authentication failure | `journalctl -u sshd -n 20` | Wrong credentials or key mismatch |
| Permission denied (publickey) | `sshd_config` | Password authentication may be disabled |
| Guacamole SSH shows blank screen | `podman logs guacd --tail 20` | Check guacd connectivity to target port 22 |
