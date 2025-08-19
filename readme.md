## open source AI inference framework – setup guide

This guide describes how to start the project locally or on a server with Docker Compose. It follows the principle:

- **.env (changeable at runtime)**: Values that can change frequently and are relevant at runtime.
- **docker-compose.yaml (usually fixed)**: Values that do not usually need to be changed.

### Requirements

- Docker Desktop or Docker Engine with Docker Compose
- Free ports: 80, 81, 443, 8080, 5678, 9000, 9001, 19530, 9091
- Windows PowerShell, macOS Terminal, or Linux Shell

### Directory structure (relevant)

- `agent-framework/`
- `docker-compose.yaml` (Starts Nginx Proxy Manager, n8n, OpenWebUI, and includes Milvus)
  - `.env-template` (template for runtime variables)
- `milvus/`
  - `docker-compose.yaml` (etcd, MinIO, Milvus Standalone – included from Agent Compose)

### TL;DR – Quick start

1) Change to the project directory and go to the `agent-framework` subfolder.
2) Copy `.env-template` to `.env` and adjust variables.
3) Start Compose.

Example:

```bash
cd agent-framework
copy .env-template .env   # Windows PowerShell: Copy-Item .env-template .env
docker compose up -d
```

### Step 1: Create `.env` from template

Go to the `agent-framework` folder and copy the template:

```bash
# Windows (PowerShell)
Copy-Item .env-template .env

# macOS/Linux
cp .env-template .env
```

Then adjust the values in `.env`. These variables are intended for runtime and can be adjusted as needed.

### Step 2: Set runtime variables in `.env`

The file `agent-framework/.env-template` contains commented defaults. Important areas:

- **OpenWebUI**
  - `OPENWEBUI_AUTH` (TRUE/FALSE): Force login.
- `OPENWEBUI_ENABLE_LOGIN_FORM` (TRUE/FALSE): Enable/disable login form (e.g., disable for SSO).
  - `MILVUS_URI`: Default `http://milvus-standalone:19530` – hostname corresponds to the Milvus service.
  - `MILVUS_TOKEN`: Default `root:Milvus`.
  - `MILVUS_DB`: Default `default`.
  - `VECTOR_DB`: Default `milvus`.

- **n8n**
  - `DOMAIN_NAME`: Your domain, e.g. `example.com`.
  - `SUBDOMAIN`: Subdomain for n8n, e.g. `n8n` → n8n would then be accessible at `https://n8n.example.com`.
  - `GENERIC_TIMEZONE`: Time zone, e.g. `Europe/Berlin`.
  - `SSL_EMAIL`: Email for certificate processes (may be required by your setup/proxy).
  - `N8N_SECURE_COOKIE`: Must be set to `TRUE` in production.

Note: The `.env` values are read in `agent-framework/docker-compose.yaml` via `environment:`.

### Step 3: Understanding Compose files (usually fixed)

- `agent-framework/docker-compose.yaml` starts:
- `nginx-proxy-manager` (ports 80, 81, 443)
  - `n8n` (port 5678) – uses `N8N_HOST`, `WEBHOOK_URL`, `TZ`, `N8N_SECURE_COOKIE`, among others
  - `open-webui` (port 8080) – uses Milvus variables from `.env`
  - Finally, includes the file `../milvus/docker-compose.yaml` via `include`

- `milvus/docker-compose.yaml` starts:
  - `etcd`
  - `minio` (ports 9000, 9001)
  - `milvus-standalone` (ports 19530, 9091)

These Compose files are prepared in such a way that you normally do not need to change them. Just make sure that the ports mentioned are free on your system.

### Step 4: Start services

Start all services from the `agent-framework` folder:

```bash
cd agent-framework
docker compose up -d
```

When starting for the first time, images are loaded and volumes/folders are created automatically:

- `./docker-volumes/npm` and `./docker-volumes/npm/letsencrypt` for Nginx Proxy Manager
- `n8n_data` (named volume) for n8n
- `./docker-volumes/open-webui` for OpenWebUI
- `./milvus/volumes/*` for etcd/MinIO/Milvus


### Step 5: Access the services

- **Nginx Proxy Manager (NPM)**: `http://<your-host-ip>:81`
  - For the default login, see the NPM documentation; change the user name/password immediately.
  - You can optionally terminate public domains/SSL certificates via NPM.

- **n8n**: Depending on your setup, either directly `http://<your-host-ip>:5678` or via your domain `https://<SUBDOMAIN>.<DOMAIN_NAME>`

- **OpenWebUI**: `http://<your-host-ip>:8080`

- **MinIO Console**: `http://<your-host-ip>:9001` (default credentials set in Milvus Compose: `minioadmin` / `minioadmin`)

- **Milvus**: gRPC/HTTP ports: 19530/9091 (for clients/integrations)

### Making changes at runtime

If you change `.env` values, restart the affected services, e.g.:

```bash
docker compose restart n8n
docker compose restart open-webui
```

Or restart them:

```bash
docker compose up -d n8n open-webui
```

### Stopping and removing

```bash
# Stop
docker compose down

# Stop + remove volumes (data) – Caution: Data loss!
docker compose down -v
```

### Troubleshooting

- **Ports occupied**: Adjust port mappings in the Compose files if necessary (only if absolutely necessary). By default, they are fixed.
- **Compose include missing**: Make sure that your Docker Compose version supports the `include` feature (Docker Compose v2.x). Alternatively, you could start the Milvus services separately from the `milvus` folder.
- **Milvus connection in OpenWebUI**: Check `MILVUS_URI`, `MILVUS_TOKEN`, `MILVUS_DB` in `.env`. The default URI `http://milvus-standalone:19530` corresponds to the service name from the Compose definition.
- **n8n cookies/SSL**: In production, set `N8N_SECURE_COOKIE=TRUE` and run n8n behind HTTPS (e.g., via Nginx Proxy Manager with a real certificate).
