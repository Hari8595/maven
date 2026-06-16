# Production Run Instructions —  Docker Stack

Step-by-step guide to run the hardened stack (frontend + backend + MySQL + Redis) in production on your EC2, keep it running 24/7, load your real database dump, and verify it works.

---

## 1. Prerequisites (one-time on the server)

Confirm Docker + Compose (official install) are present:

```bash
docker --version
docker compose version
```

If missing, install the official Docker Engine + Compose plugin (Docker's documented method), then:

```bash
sudo usermod -aG docker $USER
```

Log out and back in so the group change applies (run docker without `sudo` after this).

---

## 2. Put the files in place

Create the project structure and copy in your code:

```
prod/
├── docker-compose.yml          ← from the config file
├── .env                        ← you create this (step 3)
├── .env.example                ← from the config file
├── backend/   (Dockerfile + .dockerignore + your real backend code)
├── frontend/  (Dockerfile + .dockerignore + your real frontend code)
└── db/init/00-create-databases.sql
```

Copy each block from `PRODUCTION-DOCKER-CONFIG.md` into the matching file.

**Important:** drop your **real** Employment 360 / Recruitment 360 code into `backend/` and `frontend/` (not sample code). The Dockerfiles wrap your code.

---

## 3. Create and secure the `.env`

```bash
cp .env.example .env
nano .env          # fill in every value
```

Generate strong secrets:

```bash
openssl rand -base64 32      # run once per password/secret
```

Lock the file down (CIS — restrict secret file permissions):

```bash
chmod 600 .env
```

Set `MYSQL_DATABASE` to the database name inside your real dump — check it:

```bash
head -40 your-dump.sql        # look for CREATE DATABASE / USE
```

---

## 4. Point the backend Dockerfile at the right entry file

Open `backend/Dockerfile`, check the last line:

```dockerfile
CMD ["node", "src/index.js"]
```

Change `src/index.js` to your real entry file if different. Find it with:

```bash
cat backend/package.json | grep -E '"main"|"start"'
```

---

## 5. Build and start (24/7)

```bash
docker compose up -d --build
```

First build pulls images and compiles the apps — give it a few minutes.

Because every service has `restart: always`, the stack now runs 24/7: it restarts on crash and comes back automatically after a server reboot.

---

## 6. Verify everything is healthy

```bash
docker compose ps
```

Wait until `mysql` and `redis` show **healthy** and `backend`/`frontend` show **up/running**.

Then check the endpoints (from the server):

```bash
curl http://localhost:5000/health          # backend → expect {"status":"ok"}
curl -I http://localhost:3000              # frontend → expect HTTP 200
```

Check logs if anything is off:

```bash
docker compose logs backend --tail 40
docker compose logs mysql --tail 40
```

---

## 7. Load your real SQL dump

The init script creates the empty databases; your dump loads tables + data.

```bash
# Copy your dump to the server first (from your laptop, PowerShell):
#   scp -i "king.pem" your-dump.sql ubuntu@<server>:~/allzone-prod/

# Then load it into the running MySQL container:
docker compose exec -T mysql \
  mysql -u root -p"$MYSQL_ROOT_PASSWORD" emp360_db < your-dump.sql
```

(Replace `emp360_db` with the database name from step 3 if different.)

Verify it loaded:

```bash
docker compose exec mysql \
  mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "USE emp360_db; SHOW TABLES; SELECT COUNT(*) FROM employees;"
```

---

## 8. View it securely in a browser (no public exposure)

The services are bound to `127.0.0.1` on the server — not open to the internet. To view from your laptop, use an SSH tunnel (Windows PowerShell):

```powershell
ssh -i "king.pem" -L 3000:localhost:3000 -L 5000:localhost:5000 ubuntu@<your-ec2-host>
```

Then open `http://localhost:3000` in your browser — it tunnels securely to the server. This is the safe way to demo without opening ports publicly.

For real public access later, put an **ALB (or reverse proxy) + HTTPS** in front — don't bind the containers to `0.0.0.0`.

---

## 9. Day-to-day operations

```bash
docker compose ps                    # status
docker compose logs -f backend       # follow logs
docker compose restart backend       # restart one service
docker compose down                  # stop all (data persists in volumes)
docker compose up -d --build         # rebuild + start after a code change
docker compose pull                  # update base images, then up -d
```

**Update flow (new code):**
```bash
git pull                             # get latest code
docker compose up -d --build         # rebuild changed images, restart
```

---

## 10. Prove 24/7 self-healing (good demo moment)

```bash
docker compose kill backend          # simulate a crash
docker compose ps                    # backend is down
# wait ~10 seconds
docker compose ps                    # it restarted itself
```

Test reboot survival:

```bash
sudo reboot
# reconnect after a minute:
docker compose ps                    # everything back up automatically
```

---

## 11. Backups (don't skip in production)

Your live data is in the `mysql_data` volume. Take regular dumps and store them **off the server** (encrypted S3):

```bash
# daily dump
docker compose exec -T mysql \
  mysqldump -u root -p"$MYSQL_ROOT_PASSWORD" --all-databases | gzip > backup-$(date +%F).sql.gz

# then upload to private encrypted S3 (server reaches it via the S3 endpoint)
aws s3 cp backup-$(date +%F).sql.gz s3://your-private-backup-bucket/ --sse aws:kms
```

Automate this with a daily `cron`/systemd timer.

---

## 12. Troubleshooting (common first-run issues)

| Symptom | Cause | Fix |
|---|---|---|
| `port is already allocated` | Something else uses 3000/5000/3306 | Stop it, or change the host port in compose |
| backend keeps restarting | wrong entry file, or DB not ready | check `docker compose logs backend`; fix `CMD` (step 4) |
| `read-only file system` error | app writes to disk | remove `read_only: true` for that service, or mount a volume for the write path |
| MySQL `Access denied` | password mismatch / stale volume | `docker compose down -v` to reset (⚠️ wipes data), recheck `.env` |
| native module build fails | missing build deps in Alpine | add `RUN apk add --no-cache python3 make g++` in the build stage |
| backend can't reach Twilio | `backend-net` is `internal: true` | move backend to a non-internal network for outbound |

---

## 13. Security reminders (CIS / OWASP)

- `.env` is `chmod 600` and **gitignored** — never commit it.
- Secrets stay backend-side; **never** behind `NEXT_PUBLIC_`.
- MySQL bound to `127.0.0.1` only; Redis not exposed at all.
- Containers run **non-root**, read-only, with dropped capabilities.
- Don't `cat .env` or run `docker inspect` on a shared screen — those reveal secrets.
- For the strongest posture, move secrets to **AWS Secrets Manager** (injected at deploy) instead of an `.env` file on disk.
