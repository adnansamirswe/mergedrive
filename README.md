# MergeDrive

A self-hosted storage gateway that pools multiple Google Drive accounts behind one clean REST API. Connect N drives, route uploads to whichever has space, and let your apps upload, list and download files with scoped API keys.

**Live Demo:** [drivemommy.bmgsl.com](https://megadrive.bmgsl.com)

## Why

Google gives you 15 GB per account. Most projects only use one. MergeDrive lets you treat multiple Drive accounts as a single storage pool — files stream directly to Drive, nothing touches your server disk.

## Features

- **Multi-drive pooling** — connect unlimited Google Drive accounts via OAuth
- **Smart upload routing** — most-available, round-robin, or priority-based
- **Scoped API keys** — `files:read`, `files:upload`, `files:download`, `files:delete`, `storage:read`, `accounts:read`
- **File sync** — index Drive files into Postgres for instant search
- **Virtual folders** — app-level organization mapped onto a Drive folder
- **Dashboard** — manage files, drives, API keys, and routing policy
- **Zero disk usage** — files stream from your server to Google Drive, never stored locally

## Stack

| Layer | Tech |
|-------|------|
| Frontend | Next.js, Tailwind CSS, shadcn/ui |
| API | Bun, Hono |
| Database | Postgres, Prisma |
| Storage | Google Drive API |

## Quick start (Docker)

```bash
git clone https://github.com/adnansamirswe/mergedrive.git
cd mergedrive
cp .env.example .env
```

Edit `.env` with your Postgres URL and secrets:

```bash
openssl rand -hex 32   # JWT_ACCESS_SECRET
openssl rand -hex 16   # TOKEN_ENCRYPTION_KEY
```

Start everything:

```bash
docker compose up -d
```

The API runs on `:3000`, the web dashboard on `:3001`.

## Deploy on a VPS

### 1. Install Docker

```bash
curl -fsSL https://get.docker.com | sh
```

### 2. Clone and configure

```bash
git clone https://github.com/adnansamirswe/mergedrive.git
cd mergedrive
cp .env.example .env
```

Edit `.env`:

```env
DATABASE_URL=postgresql://postgres:yourpassword@db:5432/mergedrive
JWT_ACCESS_SECRET=<random-32-chars>
TOKEN_ENCRYPTION_KEY=<random-32-hex-chars>
FRONTEND_URL=https://mergedrive.yourdomain.com
GOOGLE_REDIRECT_URI=https://mergedrive.yourdomain.com/api/connected-accounts/google/callback
NEXT_PUBLIC_API_URL=https://mergedrive.yourdomain.com/api
```

### 3. Start with Docker Compose

```bash
docker compose up -d
```

This starts three containers:
- **db** — Postgres 16
- **api** — Bun + Hono on port 3000
- **web** — Next.js on port 3000 (map to 3001 on host)

### 4. Set up a reverse proxy

Point your domain to the VPS, then add a reverse proxy (Nginx, Caddy, or Traefik) to route traffic:

| Path | Target |
|------|--------|
| `https://yourdomain.com/api/*` | API container port 3000 |
| `https://yourdomain.com/*` | Web container port 3000 |

**Caddy** (auto-HTTPS):

```
mergedrive.yourdomain.com {
    handle /api/* {
        reverse_proxy api:3000
    }
    handle {
        reverse_proxy web:3000
    }
}
```

**Nginx**:

```nginx
server {
    listen 443 ssl;
    server_name mergedrive.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/mergedrive.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mergedrive.yourdomain.com/privkey.pem;

    location /api/ {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        client_max_body_size 5G;
    }

    location / {
        proxy_pass http://127.0.0.1:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 5. Google OAuth setup

1. Go to [Google Cloud Console](https://console.cloud.google.com/apis/credentials)
2. Create an OAuth 2.0 Client ID (Web application)
3. Add authorized redirect URIs:
   ```
   https://mergedrive.yourdomain.com/api/auth/google/callback
   https://mergedrive.yourdomain.com/api/connected-accounts/google/callback
   ```
4. Open the dashboard, go to **Settings**, and save your Google Client ID and Secret

### 6. Connect your drives

Open the dashboard, go to **Drives**, and click **Connect Google Drive**. Repeat for each account you want to pool.

## API usage

Create an API key in the dashboard, then:

```bash
# Upload a file
curl -X POST https://mergedrive.yourdomain.com/api/v1/uploads \
  -H "Authorization: Bearer dm_live_..." \
  -F "file=@report.pdf"

# List files
curl https://mergedrive.yourdomain.com/api/v1/files \
  -H "Authorization: Bearer dm_live_..."

# Get storage quota
curl https://mergedrive.yourdomain.com/api/v1/storage/summary \
  -H "Authorization: Bearer dm_live_..."
```

## Environment variables

| Variable | Description | Default |
|----------|-------------|---------|
| `DATABASE_URL` | Postgres connection string | — |
| `APP_PORT` | API listen port | `3000` |
| `FRONTEND_URL` | Dashboard URL for OAuth redirects | `http://localhost:3000` |
| `JWT_ACCESS_SECRET` | Secret for signing access tokens | — |
| `TOKEN_ENCRYPTION_KEY` | 32 hex chars for encrypting Drive tokens | — |
| `GOOGLE_REDIRECT_URI` | OAuth callback URL | — |
| `NEXT_PUBLIC_API_URL` | API URL the frontend calls | `http://localhost:3000` |
| `PORT` | Web listen port | `3000` |
| `MAX_UPLOAD_BYTES` | Max upload size in bytes | `5368709120` (5 GB) |

## License

MIT — bring your own Google Cloud project.
