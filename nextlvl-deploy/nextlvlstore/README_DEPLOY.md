# NextLvlStore — VPS Deployment Guide

A self-contained Node.js application. No build step required — the server
and frontend are already compiled and ready to run.

---

## Requirements

- **Node.js** v18 or higher (v20 LTS recommended)
- **PostgreSQL** 14 or higher
- Linux VPS (Ubuntu 20.04 / Debian 11 or newer)

---

## 1. Install Node.js (if not installed)

```bash
# Using NodeSource (recommended)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verify
node --version   # should print v20.x.x
```

---

## 2. Install & Start PostgreSQL

```bash
sudo apt install -y postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Create database and user
sudo -u postgres psql -c "CREATE USER nlvuser WITH PASSWORD 'yourpassword';"
sudo -u postgres psql -c "CREATE DATABASE nextlvlstore OWNER nlvuser;"
```

---

## 3. Upload & Extract the App

```bash
# On your VPS
mkdir -p /opt/nextlvlstore
cd /opt/nextlvlstore

# Upload nextlvl-deploy.zip via SCP or SFTP, then:
unzip nextlvl-deploy.zip
```

---

## 4. Configure Environment Variables

```bash
cp .env.example .env
nano .env   # Edit with your values
```

Required values to fill in:

| Variable       | Description                                          |
|---------------|------------------------------------------------------|
| `DATABASE_URL` | PostgreSQL connection string                         |
| `PORT`         | Port to listen on (e.g. 3000)                       |
| `NODE_ENV`     | Must be `production`                                 |
| `JWT_SECRET`   | Long random secret for token signing                 |

Generate a secure JWT_SECRET:
```bash
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

---

## 5. Start the App

```bash
# Load .env and start
export $(cat .env | grep -v '#' | xargs)
node dist/index.mjs
```

Or use the npm script:
```bash
# Load env file manually first, then:
npm start
```

The app will:
- Automatically create the required database tables on first start
- Create the default admin account: **lboss / admin123**

Open your browser: `http://your-server-ip:3000`

---

## 6. Run as a Background Service (Recommended)

### Option A — PM2 (process manager)

```bash
npm install -g pm2

# Start with env file
pm2 start dist/index.mjs --name nextlvlstore --env-file .env \
  --node-args="--env-file=.env"

# Or pass env vars directly:
pm2 start dist/index.mjs --name nextlvlstore \
  --env NODE_ENV=production \
  --env PORT=3000 \
  --env DATABASE_URL="postgresql://nlvuser:yourpassword@localhost:5432/nextlvlstore" \
  --env JWT_SECRET="your_secret_here"

pm2 save               # Save process list
pm2 startup            # Auto-start on reboot
```

### Option B — systemd service

```bash
sudo nano /etc/systemd/system/nextlvlstore.service
```

Paste:
```ini
[Unit]
Description=NextLvlStore Dashboard
After=network.target postgresql.service

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/nextlvlstore
EnvironmentFile=/opt/nextlvlstore/.env
ExecStart=/usr/bin/node dist/index.mjs
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable nextlvlstore
sudo systemctl start nextlvlstore
sudo systemctl status nextlvlstore
```

---

## 7. (Optional) Nginx Reverse Proxy

If you want the app on port 80/443:

```bash
sudo apt install -y nginx

sudo nano /etc/nginx/sites-available/nextlvlstore
```

Paste:
```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_cache_bypass $http_upgrade;
        client_max_body_size 20m;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/nextlvlstore /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

For HTTPS (free SSL):
```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d your-domain.com
```

---

## 8. Default Login

| Field    | Value     |
|----------|-----------|
| Username | `lboss`   |
| Password | `admin123` |

**Change your password immediately** after first login via **Settings → Change Password**.

---

## File Structure

```
nextlvlstore/
├── dist/                   # Compiled server (Node.js, self-contained)
│   ├── index.mjs           # Main entry point — run this
│   ├── pino-worker.mjs     # Logging worker
│   ├── pino-file.mjs       # Logging worker
│   ├── pino-pretty.mjs     # Logging worker
│   └── thread-stream-worker.mjs
├── public/                 # React frontend (static files served by Express)
│   ├── index.html
│   └── assets/
├── package.json
├── .env.example            # Template — copy to .env
└── README_DEPLOY.md        # This file
```

---

## Troubleshooting

**App won't start:**
- Check `DATABASE_URL` is correct and PostgreSQL is running
- Ensure `NODE_ENV=production` is set
- Check port is not in use: `sudo lsof -i :3000`

**Database connection failed:**
- Test connection: `psql "postgresql://nlvuser:yourpassword@localhost:5432/nextlvlstore"`
- Check PostgreSQL is running: `sudo systemctl status postgresql`

**Login page not loading:**
- Ensure `NODE_ENV=production` — this enables static file serving
- Check Node.js version: `node --version` (must be v18+)

**Forgot admin password:**
```bash
# Reset via PostgreSQL
psql "$DATABASE_URL" -c "UPDATE users SET password_hash = '\$2b\$10\$...' WHERE username = 'lboss';"
# Or delete the user row and restart the app (it will re-create the default admin)
psql "$DATABASE_URL" -c "DELETE FROM users WHERE username = 'lboss';"
# Then restart the app — lboss/admin123 will be re-created
```
