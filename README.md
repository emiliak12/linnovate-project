# Linnovate Exercise

---

## Table of Contents
1. [Prerequisites](#1-prerequisites)
2. [Goals](#2-goals)
3. [Steps](#3-steps)
4. [Challenges & Errors](#4-challenges--errors)
5. [Variables](#5-variables)

---

## 1. Prerequisites

### System
- RHEL 8 server

### Required Packages

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo dnf install git vim yum-utils certbot net-tools
sudo wget unzip
```

---

## 2. Goals

### Rocketchat
- Configure Rocketchat with MongoDB
- Configure SSL certificate and SSO (Google OAuth)
- Configure monitoring (Grafana + Loki)
- Configure SMTP server for mailing functionality

### Jitsi
- Deploy Jitsi Meet using official Docker images
- Configure SSL via Traefik reverse proxy
- Implement Google SSO via jitsi-openid service

### Security
- Use rootless containers (Podman Compose) for enhanced security
- Exclude `.env` files from Git (gitignore) to protect secrets
- Use sha instead of tags

---



---

## 3. Steps

### Step 1 — Basic Rocketchat Configuration
- Configure MongoDB and Rocketchat via Docker Compose
- Set up Traefik with LetsEncrypt for SSL (following official Rocketchat docs)
- Install certbot and required SSL packages
- Review configuration files for exposed secrets and apply security best practices
- Create SSH key and push initial configuration + README to GitHub

### Step 2 — Jitsi Configuration
- Follow official [Jitsi Docker documentation](https://jitsi.github.io/handbook/docs/devops-guide/devops-guide-docker/)
- Generate required scripts and store values in `.env` file
- Add `.env` to `.gitignore`; document setup in README
- Remove unnecessary files from repo: `release.sh`, `Makefile`, `base/`, `resources/`, `CHANGELOG.md`

### Step 3 — Monitoring
- Copy and adapt Grafana configuration
- Verify Grafana accessible at `localhost:5050/grafana` from inside VM
- Fix external access by binding Grafana to `0.0.0.0`
- Integrate Loki; remove unneeded opentelemetry configuration

### Step 4 — Security & Authentication
- Register subdomains on DuckDNS: `chat-linnovate.duckdns.org` / `video-linnovate.duckdns.org`
- Configure Traefik as reverse proxy and load balancer for SSL termination
- Connect Jitsi web container to `rocketchat_default` network (`external: true`)
- Add Jitsi labels and backend configuration to Traefik manifests (`services.tpl`)
- Configure Google SSO for Rocketchat ([official OAuth guide](https://docs.rocket.chat/docs/google-oauth-setup))
- Configure Google SSO for Jitsi using [jitsi-openid](https://github.com/MarcelCoding/jitsi-openid)
- Add SMTP configuration for Rocketchat notifications

---

## 4. Challenges & Errors

### Infrastructure — VM Stability

The Pluralsight playground VM resets its IP every 4 hours. Attempts to use a local VMware/CentOS VM failed due to BIOS settings and insufficient disk space. AWS EC2 micro and Oracle Cloud free tier were both too limited and crashed under load. Resolved by staying on Pluralsight and using DuckDNS dynamic IP updates to keep subdomains current.

#### Fix
Update DuckDNS record with the new IP on each VM restart:

```bash
curl 'https://www.duckdns.org/update?domains=chat-linnovate&token=<TOKEN>&ip='
```

---

### Traefik — Port 8080 Already Allocated

Traefik dashboard failed to start because port 8080 was already in use. Changed the dashboard port in the `.env` file. After restart, SSL certificates were issued successfully.

#### Fix

In `.env`, change the Traefik dashboard port:

```env
TRAEFIK_DASHBOARD_PORT=8081
```

Then restart:

```bash
docker compose down && docker compose up -d
```

---

### Rocketchat — Stale Site URL Cached in MongoDB

After reconfiguring the subdomain, Rocketchat continued using the old placeholder URL (`chat.yourdomain.com`) because the value was cached in MongoDB.

#### Fix

Query the cached value:

```bash
docker exec -it <mongo_container> mongosh rocketchat --eval \
  'db.rocketchat_settings.findOne({_id:"Site_Url"})'
```

Update it to the correct subdomain:

```bash
docker exec -it <mongo_container> mongosh rocketchat --eval \
  'db.rocketchat_settings.updateOne({_id:"Site_Url"},{$set:{value:"https://chat-linnovate.duckdns.org"}})'
```

Then restart the Rocketchat stack:

```bash
docker compose down && docker compose up -d
```

---

### Jitsi — LetsEncrypt Certificate Failure

- Typo in `.env`: `LETSENCRYPT_DOMAIN` set to `.com` instead of `.org`
- Port 80 was occupied by Traefik (used by Rocketchat)
- Resolution: routed Jitsi through Traefik for SSL termination instead of using LetsEncrypt directly

#### Fix

Correct the `.env` typo:

```env
LETSENCRYPT_DOMAIN=video-linnovate.duckdns.org
```

Connect the Jitsi web container to the Traefik network in `docker-compose.yml`:

```yaml
networks:
  - rocketchat_default
```

Declare it as external:

```yaml
networks:
  rocketchat_default:
    external: true
```

---

### Jitsi — Traefik Configuration Incomplete

Jitsi was not appearing in Traefik routing because the `services.tpl` file (which defines backend reach) was not updated. Also enabled debug logging to diagnose the issue.

#### Fix

Add Jitsi backend entry to `/traefik_config/services.tpl`:

```toml
[http.services.jitsi.loadBalancer]
  [[http.services.jitsi.loadBalancer.servers]]
    url = "http://jitsi-web-1:80"
```

Enable Traefik debug logging in the compose `command` field for troubleshooting:

```yaml
command:
  - "--log.level=DEBUG"
  - "--accesslog=true"
```

---

### Jitsi — SSL Appeared Broken (Browser Cache)

After all fixes, the site still showed as insecure. All Traefik logs confirmed the correct certificate was being served. Testing in an incognito window confirmed the issue was browser-level caching.

#### Fix

Hard-clear browser cache , or verify in incognito mode/another browser first to isolate cache issues before deeper debugging.

---

### Jitsi — Google SSO 404 Error

After deploying jitsi-openid, a 404 error appeared. Traefik middleware was redirecting traffic to the wrong location. Fixed by correcting the middleware routing configuration so requests were forwarded correctly to the jitsi-openid service in traefik configuration.

#### Fix

Add the correct Traefik labels to the `jitsi-openid` container in `docker-compose.yml`:

```yaml
              middlewares:
                - strip-jitsi-openid

          middlewares:
            strip-jitsi-openid:
              stripPrefix:
                prefixes:
                  - "/jitsi-openid"
```

## 5. Variables
---

### Rocket.Chat `.env`

#### Application

| Variable | Value | Description |
|---|---|---|
| `REG_TOKEN` | _(empty)_ | Optional Rocket.Chat Cloud registration token. Leave empty for self-hosted without cloud integration. |
| `DOMAIN` | `chat-linnovate.duckdns.org` | The bare domain (no protocol, no trailing slash) used by Traefik routing rules to match incoming requests. |
| `ROOT_URL` | `https://chat-linnovate.duckdns.org` | The full public URL of Rocket.Chat. Must include protocol. Stored in MongoDB on first run — if changed after first boot, must be updated directly in the database. |
| `RELEASE` | `8.0.1` | Rocket.Chat Docker image tag. Pins the version to avoid unexpected upgrades on container restart. |

### SSL / Let's Encrypt

| Variable | Value | Description |
|---|---|---|
| `LETSENCRYPT_ENABLED` | `true` | Enables Traefik's ACME TLS challenge to automatically obtain and renew SSL certificates from Let's Encrypt. Requires ports 80 and 443 to be open and DNS to resolve correctly before starting. |
| `LETSENCRYPT_EMAIL` | `emily.kapelevich@gmail.com` | Email address sent to Let's Encrypt for certificate expiry notifications. Required when `LETSENCRYPT_ENABLED=true`. |
| `TRAEFIK_PROTOCOL` | `https` | Tells Traefik which config file to load — `http/dynamic.yml` or `https/dynamic.yml`. Set to `https` for production. |

### Traefik

| Variable | Value | Description |
|---|---|---|
| `TRAEFIK_HTTP_PORT` | `80` | Host port bound for HTTP traffic. Required to be open for Let's Encrypt HTTP challenge and HTTP→HTTPS redirect. |
| `TRAEFIK_HTTPS_PORT` | `443` | Host port bound for HTTPS traffic. Must be open externally for SSL to work. |
| `TRAEFIK_DASHBOARD_PORT` | `8081` | Host port for the Traefik dashboard UI. Changed from the default `8080` because that port was already allocated on the host. Only accessible locally. |

### Grafana

| Variable | Value | Description |
|---|---|---|
| `GRAFANA_DOMAIN` | _(empty)_ | If set, Grafana is served at its own subdomain. Left empty to use path-based access via `GRAFANA_PATH` instead. |
| `GRAFANA_PATH` | `/grafana` | Subpath under the main domain where Grafana is served. Accessible at `https://chat-linnovate.duckdns.org/grafana`. |
| `GRAFANA_ADMIN_PASSWORD` | *secretstring* | Password for the Grafana `admin` user. Change before deploying to any non-local environment. |
| `GRAFANA_HOST_PORT` | `5050` | Host port Grafana is mapped to for direct local access (bypassing Traefik). |
| `GRAFANA_BIND_IP` | `0.0.0.0` | Changed from the default `127.0.0.1` to allow external access through Traefik. Without this change Grafana only accepts connections from localhost. |

### Prometheus

| Variable | Value | Description |
|---|---|---|
| `PROMETHEUS_PORT` | `9000` | Host port for Prometheus UI. Set to `9000` instead of the default `9090` because `9090` conflicts with Cockpit on RHEL 8. |
| `PROMETHEUS_RETENTION_SIZE` | `15GB` | Maximum disk space Prometheus will use before dropping old data. |
| `PROMETHEUS_RETENTION_TIME` | `15d` | Maximum age of metrics data before it is dropped. |

### MongoDB

| Variable | Value | Description |
|---|---|---|
| `MONGODB_BIND_IP` | `127.0.0.1` | Restricts MongoDB to loopback only. Keeps the database inaccessible from outside the host. Do not change to `0.0.0.0` without adding authentication. |
| `MONGODB_PORT_NUMBER` | `27017` | Standard MongoDB port. If running multiple MongoDB instances on the same host, change one to avoid port conflicts. |

### NATS

| Variable | Value | Description |
|---|---|---|
| `NATS_PORT_NUMBER` | `4222` | Standard NATS client port. Used internally by Rocket.Chat for message transport. |
| `NATS_BIND_IP` | `127.0.0.1` | Restricts NATS to loopback only. Keep as-is — NATS does not need to be externally accessible. |

---

## Jitsi `.env`

> The Jitsi `.env` is generated by running `bash gen-passwords.sh` which populates all password fields automatically. The variables below are the ones that must be set manually.

### Application

| Variable | Value | Description |
|---|---|---|
| `PUBLIC_URL` | `https://video-linnovate.duckdns.org` | Full public URL of the Jitsi instance. Used by all Jitsi components to construct internal and external links. |
| `HTTP_PORT` | `8000` | Host port for Jitsi web HTTP. Set to a non-standard port because port 80 is held by Traefik. |
| `HTTPS_PORT` | `8443` | Host port for Jitsi web HTTPS. Jitsi's internal Nginx listens here but SSL is handled by Traefik externally. |
| `ENABLE_LETSENCRYPT` | `0` | Disables Jitsi's built-in acme.sh certificate handling. Set to `0` because port 80 is occupied by Traefik — SSL is managed by Traefik instead. |
| `TZ` | `UTC` | Timezone for all Jitsi containers. |

### Authentication (JWT + Google SSO)

| Variable | Value | Description |
|---|---|---|
| `ENABLE_AUTH` | `1` | Enables authentication on Jitsi. Without this anyone with the URL can join any room. |
| `ENABLE_GUESTS` | `0` | Disables unauthenticated guest access. All users must authenticate via Google before joining. |
| `AUTH_TYPE` | `jwt` | Sets authentication method to JSON Web Tokens. Required for the jitsi-openid Google SSO adapter to work. |
| `JWT_APP_ID` | `video-linnovate.duckdns.org` | Identifier for this Jitsi instance in JWT tokens. Must match `JITSI_SUB` in the jitsi-openid container. |
| `JWT_APP_SECRET` | _(generated)_ | Secret used to sign and verify JWT tokens. Must match `JITSI_SECRET` in the jitsi-openid container. Generated by `gen-passwords.sh`. |
| `JWT_ACCEPTED_ISSUERS` | `jitsi` | Comma-separated list of accepted JWT issuers. Must match what jitsi-openid sets in the token. |
| `JWT_ACCEPTED_AUDIENCES` | `jitsi` | Comma-separated list of accepted JWT audiences. Must match what jitsi-openid sets in the token. |
| `TOKEN_AUTH_URL` | `https://video-linnovate.duckdns.org/jitsi-openid/room/{room}` | URL Jitsi redirects unauthenticated users to. The `{room}` placeholder is replaced with the actual room name at runtime. Points to the jitsi-openid adapter which handles the Google OAuth flow. |

### Generated passwords (via gen-passwords.sh)

These are auto-generated and must not be set manually. They are shared between Jitsi components for internal authentication:

| Variable | Description |
|---|---|
| `JICOFO_AUTH_PASSWORD` | Password for JiCoFo to authenticate with Prosody XMPP |
| `JVB_AUTH_PASSWORD` | Password for Jitsi Videobridge to authenticate with Prosody |
| `JIGASI_XMPP_PASSWORD` | Password for Jigasi (SIP gateway) — generated but unused if Jigasi is not deployed |
| `JIBRI_RECORDER_PASSWORD` | Password for Jibri recorder — generated but unused if Jibri is not deployed |
| `JIBRI_XMPP_PASSWORD` | Password for Jibri XMPP connection — generated but unused if Jibri is not deployed |
| `JIGASI_TRANSCRIBER_PASSWORD` | Password for transcription service — generated but unused if transcription is not enabled |

---

## jitsi-openid (Google SSO adapter)

These are set in `docker-compose.yml` under the `jitsi-openid` service, sourced from the Jitsi `.env`:

| Variable | Value | Description |
|---|---|---|
| `JITSI_SECRET` | Same as `JWT_APP_SECRET` | Used to sign the JWT token issued to Jitsi after successful Google login. Must match exactly. |
| `JITSI_URL` | `https://video-linnovate.duckdns.org` | Base URL of the Jitsi instance. Used to construct the redirect URL after successful authentication. |
| `JITSI_SUB` | `video-linnovate.duckdns.org` | JWT subject — must match `JWT_APP_ID` in Jitsi `.env`. |
| `ISSUER_URL` | `https://accounts.google.com` | Google's OpenID Connect issuer URL. Fixed value for Google OAuth. |
| `BASE_URL` | `https://video-linnovate.duckdns.org` | Public base URL of the jitsi-openid service itself. Used to construct the callback URL. |
| `CLIENT_ID` | _(from Google Console)_ | OAuth 2.0 Client ID created in Google Cloud Console. |
| `CLIENT_SECRET` | _(from Google Console)_ | OAuth 2.0 Client Secret. Keep secret — never commit to git. |
| `CLIENT_REDIRECT_URL` | `https://video-linnovate.duckdns.org/callback` | The exact URI registered in Google Console as an authorized redirect URI. Google sends the auth code here after user login. |


