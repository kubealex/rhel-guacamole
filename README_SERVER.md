# Apache Guacamole Server on RHEL with Podman

This guide deploys Apache Guacamole 1.6.0 on RHEL using rootless Podman.

Architecture:

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

## 1. Install Podman

```bash
sudo dnf install -y podman

mkdir -p ~/guacamole
cd ~/guacamole
```

## 2. Create the container network

```bash
podman network create guac-net
```

All Guacamole components use this private network.

Only the Guacamole web interface needs to be published externally.

## 3. Generate the PostgreSQL schema

```bash
podman run --rm docker.io/guacamole/guacamole:1.6.0 \
  /opt/guacamole/bin/initdb.sh --postgresql > initdb.sql
```

Verify:

```bash
ls -lh initdb.sql
```

## 4. Start PostgreSQL

Create persistent storage:

```bash
podman volume create guac-pgdata
```

Start PostgreSQL:

```bash
podman run -d \
  --name guac-postgres \
  --network guac-net \
  -e POSTGRES_DB=guacamole_db \
  -e POSTGRES_USER=guacamole_user \
  -e POSTGRES_PASSWORD='CHANGE_ME_DB_PASSWORD' \
  -v guac-pgdata:/var/lib/postgresql/data:Z \
  -v ./initdb.sql:/docker-entrypoint-initdb.d/initdb.sql:ro,Z \
  docker.io/library/postgres:16
```

Verify:

```bash
podman logs guac-postgres
```

A successful initialization eventually reports:

```text
database system is ready to accept connections
```

The initialization scripts are executed only when the PostgreSQL data directory is empty.

## 5. Start guacd

```bash
podman run -d \
  --name guacd \
  --network guac-net \
  docker.io/guacamole/guacd:1.6.0
```

Port 4822 does not need to be exposed outside the container network.

For additional troubleshooting, guacd can be started with:

```text
-e LOG_LEVEL=debug
```

## 6. Start Guacamole

```bash
podman run -d \
  --name guacamole \
  --network guac-net \
  -e GUACD_HOSTNAME=guacd \
  -e POSTGRESQL_HOSTNAME=guac-postgres \
  -e POSTGRESQL_DATABASE=guacamole_db \
  -e POSTGRESQL_USERNAME=guacamole_user \
  -e POSTGRESQL_PASSWORD='CHANGE_ME_DB_PASSWORD' \
  -p 8080:8080 \
  docker.io/guacamole/guacamole:1.6.0
```

Verify:

```bash
podman ps
```

Expected containers:

```text
guac-postgres
guacd
guacamole
```

## 7. Access Guacamole

Open:

```text
http://<GUACAMOLE_SERVER>:8080/guacamole/
```

Default credentials:

```text
Username: guacadmin
Password: guacadmin
```

Change the default password immediately.

## 8. Create an RDP connection

Go to:

```text
Settings → Connections → New Connection
```

Configure:

```text
Name:       RHEL Workstation
Protocol:   RDP

Hostname:   <RHEL_WORKSTATION_IP_OR_HOSTNAME>
Port:       3389

Username:   rdpuser
Password:   <RDP_GATEWAY_PASSWORD>
```

The username and password configured here are the GNOME Remote Desktop gateway credentials.

They are **not** the Linux user's credentials.

If the RHEL workstation uses a self-signed RDP certificate:

```text
Ignore server certificate: enabled
```

After the RDP connection is established, GNOME Remote Desktop presents GDM. The actual Linux username and password are entered there.

Example:

```text
Guacamole
    |
    | rdpuser / RDP gateway password
    v
GNOME Remote Desktop
    |
    v
GDM
    |
    | sysadmin / Linux password
    v
GNOME Desktop
```

## 9. Persistence

Guacamole configuration and users are stored in PostgreSQL.

Inspect the persistent volume:

```bash
podman volume inspect guac-pgdata
```

The `guacamole` and `guacd` containers can therefore be recreated without losing configuration, provided the PostgreSQL volume is retained.

For rootless Podman deployments that must survive logout:

```bash
sudo loginctl enable-linger $USER
```

For production-style deployments on recent RHEL versions, consider managing the containers using Podman Quadlet/systemd.

## Troubleshooting

| Symptom | Check | Typical cause / action |
|---|---|---|
| Guacamole UI unavailable | `podman ps` | Verify the `guacamole` container is running and port 8080 is published |
| Guacamole fails to start | `podman logs guacamole` | Check PostgreSQL and guacd environment variables |
| Database errors | `podman logs guac-postgres` | Verify schema initialization and DB credentials |
| RDP connection fails immediately | `podman logs guacd --tail 50` | Check target hostname, port 3389 and RDP gateway credentials |
| RDP connection times out | Test network connectivity from the Guacamole host | Target port 3389 may not be reachable |
| Certificate validation error | Enable `Ignore server certificate` | Expected when using the self-signed GRD certificate |
| Need more RDP detail | Run guacd with `LOG_LEVEL=debug` | Inspect FreeRDP negotiation and authentication errors |
