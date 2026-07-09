# CHI Project Management Dashboard

Internal project/people dashboard for the CHI team. Built with Express + SQLite (better-sqlite3).

> **Continuing this project?** See [ARCHITECTURE.md](ARCHITECTURE.md) for the data model, API reference, frontend internals, and known limitations — this README only covers running and hosting.

---

## Running locally (no Docker)

**Prerequisites:** Node.js 18+

```bash
npm install
node server.js
```

Open http://localhost:3000. The database is created automatically at `data/dashboard.db` on first run.

To enable login protection locally:

```bash
DASHBOARD_PASS=secret node server.js
```

---

## Running locally with Docker

```bash
docker compose up --build
```

Open http://localhost:3000. The `data/` folder on your machine is mounted into the container, so the database survives rebuilds.

---

## Hosting on a VM

### 1. Copy the project to the VM

```bash
# From your machine
scp -r . user@vm-ip:/opt/chi-dashboard
# or clone from git after pushing
```

### 2. Set a password

```bash
cd /opt/chi-dashboard
cp .env.example .env
nano .env          # set DASHBOARD_PASS to something strong
```

### 3. Start with Docker Compose

```bash
docker compose up -d --build
```

The app listens on port 3000 inside the container, exposed on the same port on the host.

### 4. (Recommended) Put Nginx in front for HTTPS

Install Nginx and Certbot, then create `/etc/nginx/sites-available/chi-dashboard`:

```nginx
server {
    server_name dashboard.yourteam.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Enable and get a certificate:

```bash
ln -s /etc/nginx/sites-available/chi-dashboard /etc/nginx/sites-enabled/
certbot --nginx -d dashboard.yourteam.com
systemctl reload nginx
```

### 5. Updating

```bash
cd /opt/chi-dashboard
git pull            # or re-copy changed files
docker compose up -d --build
```

The database in `data/` is untouched by rebuilds.

---

## Environment variables

| Variable         | Default | Description                                      |
|------------------|---------|--------------------------------------------------|
| `DASHBOARD_PASS` | *(none)*| HTTP Basic Auth password. Auth disabled if unset |
| `DASHBOARD_USER` | `chi`   | HTTP Basic Auth username                         |
| `PORT`           | `3000`  | Port the server listens on                       |

---

## Data

- **Database:** `data/dashboard.db` — SQLite, created on first run
- **Snapshots:** `data/snapshots/` — DB snapshots saved from the Admin tab
- Neither folder is committed to git (see `.gitignore`)

## Default managers

Jacob Sherson and Morten Røndal Olsen are seeded as managers on first run.  
Managers can access the **Admin** tab (change log + DB snapshots) and the **Manage** panel (add/remove people and projects).
