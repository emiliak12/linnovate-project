# RocketChat Deployment
---

## Stack Overview

| Service | Image | Description |
|---|---|---|
| `rocketchat` | `registry.rocket.chat/rocketchat/rocket.chat:8.0.1` | Main chat application |
| `mongodb` | `mongodb/mongodb-community-server:8.2` | Primary database (replica set) |

---

## Files

| File | Purpose |
|---|---|
| `.env` | All environment variable defaults — edit this before deploying |
| `compose.yml` | Rocket.Chat service definition |
| `mongo-compose.yml` | MongoDB, NATS, and exporter services |
| `docker.yml` | Production overrides (healthchecks, etc.) |

---

## Prerequisites

- Docker Engine 24+
- Docker Compose plugin (`docker compose version`)
- Ports 80, 443, 3000, 8080 available on the host

---

## Quick Start

### 1. Clone and configure

```bash
git clone https://github.com/emiliak12/linnovate-project.git
cd linnovate-project
```

### 2. Edit `.env`

At minimum, set these values:

```env
DOMAIN=
ROOT_URL=https://chat.yourdomain.com
```

For HTTPS with Let's Encrypt:

```env
LETSENCRYPT_ENABLED=true
LETSENCRYPT_EMAIL=you@yourdomain.com
TRAEFIK_PROTOCOL=https
```

### 3. Start all services

```bash
docker compose -f compose.yml -f mongo-compose.yml up -d
```

To include production healthchecks:

```bash
docker compose -f compose.yml -f mongo-compose.yml -f docker.yml up -d
```

### 4. Access Rocket.Chat

Open your browser at `http://localhost:3000` (or the `ROOT_URL` you configured).

---

## Configuration Reference

### Rocket.Chat

| Variable | Default | Description |
|---|---|---|
| `RELEASE` | `8.0.1` | RocketChat image tag |
| `ROOT_URL` | `http://chat.yourdomain.com` | Public URL of your instance |
| `HOST_PORT` | `3000` | Host port for the web UI |
| `METRICS_PORT` | `9458` | Prometheus metrics endpoint port |
| `REG_TOKEN` | _(empty)_ | Cloud registration token |

### MongoDB

| Variable | Default | Description |
|---|---|---|
| `MONGODB_VERSION` | `8.2` | MongoDB image tag |
| `MONGODB_PORT_NUMBER` | `27017` | MongoDB port |
| `MONGODB_BIND_IP` | `127.0.0.1` | localhost |
| `MONGODB_HOST_PATH` | `mongodb_data` (volume) | Data directory or named volume |


