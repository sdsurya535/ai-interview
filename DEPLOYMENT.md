# Hostinger VPS Deployment Guide — Interview AI

## Project Structure

```
interview-ai-yt/
├── .env                   ← Root secrets for docker-compose (DO NOT COMMIT)
├── .gitignore
├── docker-compose.yml
├── Backend/
│   ├── Dockerfile
│   ├── .dockerignore
│   └── ...
└── Frontend/
    ├── Dockerfile
    ├── .dockerignore
    ├── nginx.conf
    └── ...
```

---

## Step 1 — Access Your Hostinger VPS

1. Go to [hpanel.hostinger.com](https://hpanel.hostinger.com) → **VPS** → your server
2. Note your **VPS IP address** (e.g. `123.45.67.89`)
3. SSH into it:

```bash
ssh root@123.45.67.89
```

> On first login it will ask you to set/change the root password.

---

## Step 2 — Install Docker

```bash
# Update system
apt update && apt upgrade -y

# Install Docker (official one-liner)
curl -fsSL https://get.docker.com | sh

# Allow running docker without sudo
usermod -aG docker $USER
newgrp docker

# Verify
docker --version
docker compose version
```

---

## Step 3 — Open Firewall Ports on Hostinger

In **hpanel → VPS → Firewall**, add these inbound rules:

| Port | Protocol | Purpose          |
| ---- | -------- | ---------------- |
| 22   | TCP      | SSH              |
| 80   | TCP      | Frontend (nginx) |
| 3000 | TCP      | Backend API      |

---

## Step 4 — Push Your Code to GitHub

On your **local machine** (make sure `.env` files are gitignored):

```bash
cd interview-ai-yt
git add .
git commit -m "add docker setup"
git push origin main
```

---

## Step 5 — Clone the Repo on the VPS

```bash
# On the VPS
git clone https://github.com/your-username/interview-ai-yt.git
cd interview-ai-yt
```

---

## Step 6 — Create the `.env` File on the VPS

```bash
nano .env
```

Paste and fill in your real values:

```env
# MongoDB
MONGO_ROOT_USER=admin
MONGO_ROOT_PASSWORD=YourStrongPasswordHere
MONGO_DB_NAME=interview-ai

# Backend
JWT_SECRET=lqw29H9/pFzh9cQEui/zKEDWfcG39LTHWgJFYvv8KJA=
GOOGLE_GENAI_API_KEY=your_real_gemini_api_key

# Frontend — use your VPS IP or domain
VITE_API_BASE_URL=http://123.45.67.89:3000
```

Save with `Ctrl+O`, `Enter`, then `Ctrl+X`.

---

## Step 7 — Build & Start

```bash
docker compose up --build -d
```

> ⏳ First build takes **5–10 min** (Chromium download for Puppeteer PDF generation).

---

## Step 8 — Verify Everything is Running

```bash
docker compose ps
```

You should see three containers with status `Up`:

```
NAME                  STATUS
interview_mongo       Up
interview_backend     Up
interview_frontend    Up
```

Test in browser:

- **Frontend** → `http://123.45.67.89`
- **Backend** → `http://123.45.67.89:3000`

---

## Useful Commands

```bash
docker compose logs -f                # all logs (live)
docker compose logs backend -f        # backend only
docker compose restart backend        # restart one service
docker compose down                   # stop (keeps MongoDB data)
docker compose down -v                # stop + wipe MongoDB data
```

---

## Updating the App

Every time you push new code:

```bash
# On the VPS
git pull
docker compose up --build -d
```

---

## Optional — Custom Domain + SSL (HTTPS)

If you have a domain pointed to your VPS IP:

1. On Hostinger DNS → point `A record` to your VPS IP
2. Install Certbot on the VPS:

```bash
apt install certbot -y
certbot certonly --standalone -d yourdomain.com
```

3. Update `Frontend/nginx.conf` to add HTTPS, pointing certs to `/etc/letsencrypt/live/yourdomain.com/`
4. Update root `.env`: `VITE_API_BASE_URL=https://yourdomain.com` (then rebuild frontend)

---

## Database Backup & Restore

> **⚠️ Critical Warning:** Running `docker compose down -v` **permanently deletes all your data**. The `-v` flag removes the Docker volume where MongoDB stores everything. Never run this in production unless you intend to wipe the database. Use `docker compose down` (without `-v`) to safely stop containers.

---

### Taking a Manual Backup

Run this on the VPS to dump the entire database into a compressed archive:

```bash
# Creates a backup file: backup_2026-03-04.gz
docker exec interview_mongo mongodump \
  --username admin \
  --password YourStrongPasswordHere \
  --authenticationDatabase admin \
  --db interview-ai \
  --archive \
  --gzip > backup_$(date +%Y-%m-%d).gz
```

The backup file appears in your **current directory** on the VPS. Store it safely.

---

### Restoring from a Backup

```bash
# Restore from a backup file
docker exec -i interview_mongo mongorestore \
  --username admin \
  --password YourStrongPasswordHere \
  --authenticationDatabase admin \
  --db interview-ai \
  --archive \
  --gzip < backup_2026-03-04.gz
```

> **⚠️ Warning:** Restoring overwrites existing data in the `interview-ai` database. Make sure you are restoring the correct backup file.

---

### Automated Daily Backups (Cron Job)

Set up a cron job on the VPS to auto-backup every day at 2 AM:

```bash
# Create a backup script
cat > /root/backup-mongo.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/root/mongo-backups"
mkdir -p $BACKUP_DIR

# Keep only last 7 days of backups
find $BACKUP_DIR -name "*.gz" -mtime +7 -delete

# Create today's backup
docker exec interview_mongo mongodump \
  --username admin \
  --password YourStrongPasswordHere \
  --authenticationDatabase admin \
  --db interview-ai \
  --archive \
  --gzip > $BACKUP_DIR/backup_$(date +%Y-%m-%d).gz

echo "Backup completed: $(date)"
EOF

chmod +x /root/backup-mongo.sh

# Add to crontab — runs every day at 2:00 AM
(crontab -l 2>/dev/null; echo "0 2 * * * /root/backup-mongo.sh >> /root/mongo-backups/backup.log 2>&1") | crontab -
```

**Verify the cron is set:**

```bash
crontab -l
```

---

### Download Backup to Your Local Machine

Run this on your **local machine** to copy the backup from the VPS:

```bash
scp root@123.45.67.89:/root/mongo-backups/backup_2026-03-04.gz ./
```

---

### ⚠️ Production Backup Warnings

| Risk                     | What happens                         | Prevention                                                     |
| ------------------------ | ------------------------------------ | -------------------------------------------------------------- |
| `docker compose down -v` | **Deletes ALL data permanently**     | Always use `docker compose down` (no `-v`) in production       |
| VPS disk full            | Backups fail silently                | Monitor disk: `df -h`, keep only last 7 days                   |
| No off-VPS copy          | VPS loss = data loss                 | Download backups to local machine or upload to S3/Google Drive |
| Restore without testing  | Corrupted backup discovered too late | Test restores on a staging environment periodically            |

---

> **⚠️ Important:** `VITE_API_BASE_URL` is baked into the bundle **at build time**. If you change domain/IP later, rebuild with: `docker compose up --build -d frontend`

> **🔒 Warning:** Never commit the root `.env` to Git — it contains your database password and API keys.

> **💡 Tip:** MongoDB data is stored in a Docker volume (`mongo_data`) — it survives container restarts. Only `docker compose down -v` deletes it.
