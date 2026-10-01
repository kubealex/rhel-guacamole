<p align="center">
  <img src="docs/assets/redhat-logo.svg" width="80" alt="Red Hat">
  <img src="docs/assets/avocado.svg" width="100" alt="Avocado">
  <img src="docs/assets/guacamole.svg" width="160" alt="Guacamole Bowl">
</p>

<h1 align="center">RHEL Guacamole</h1>

<p align="center">
  <strong>Browser-based remote desktop for RHEL, powered by Apache Guacamole and Podman</strong>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/RHEL-10-red?style=flat-square" alt="RHEL 10">
  <img src="https://img.shields.io/badge/Guacamole-1.6.0-green?style=flat-square" alt="Guacamole 1.6.0">
  <img src="https://img.shields.io/badge/Podman-rootless-blue?style=flat-square" alt="Podman rootless">
  <img src="https://img.shields.io/badge/RDP%20%2B%20SSH-supported-orange?style=flat-square" alt="RDP + SSH">
</p>

---

## What is this?

A complete guide to deploying [Apache Guacamole](https://guacamole.apache.org/) on RHEL using rootless Podman containers, and configuring RHEL machines as RDP and SSH targets.

Access your Linux desktops and servers from any browser — no client software needed.

```text
Browser
   |
   v
Guacamole Web App :8080
   |
   v
guacd :4822
   |
   +------ RDP :3389 -----> RHEL Workstation
   |
   +------ SSH :22 -------> RHEL Server
   |
   v
PostgreSQL
```

## Guides

| Guide | Description |
|---|---|
| [Server Setup](README_SERVER.md) | Deploy Guacamole with Podman (PostgreSQL + guacd + web app) |
| [Client Setup — RDP](README_CLIENT_RDP.md) | Configure RHEL 10 GNOME Remote Desktop for graphical sessions |
| [Client Setup — SSH](README_CLIENT_SSH.md) | Configure OpenSSH for terminal sessions |

## Quick Start

**On the Guacamole server:**

```bash
sudo dnf install -y podman
podman network create guac-net
```

Then follow the [full server guide](README_SERVER.md) to start the three containers.

**On each RHEL workstation (RDP):**

```bash
sudo dnf install -y gdm gnome-shell gnome-remote-desktop pipewire wireplumber
```

Then follow the [RDP client guide](README_CLIENT_RDP.md) to configure remote desktop access.

**On each RHEL server (SSH):**

```bash
sudo dnf install -y openssh-server
sudo systemctl enable --now sshd.service
```

Then follow the [SSH client guide](README_CLIENT_SSH.md) for firewall and hardening.

## License

This project provides documentation and configuration guides. Apache Guacamole is licensed under the [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0).
