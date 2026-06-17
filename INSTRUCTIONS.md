# Instructions — Run the Two-Application Docker Stack

How to deploy and run two applications (each frontend + backend), sharing one MySQL
(two databases) and one Redis, where **each app folder keeps its own `.env`**. Includes a
copy-paste prompt (generic, no company/app names), run steps, database loading, and
troubleshooting for every issue commonly hit.

---

## A. Copy-paste prompt (generic architecture — paste into a fresh chat)

> Use this to regenerate or adjust the setup later. It describes the architecture only — no
> company or product names — so it's safe to reuse anywhere.

```
I need a production Docker setup, hardened to OWASP and CIS standards, for the following
architecture:

- Two separate web applications. Each application has a Next.js frontend and a Node.js
  (v22) backend.
- One MySQL 8 server hosting TWO databases (one per application). BOTH backends must
  connect to BOTH databases (one application promotes/writes records into the other's
  database, and shared staff use both).
- One Redis instance, used for a message/notification queue and caching.
- Outbound integrations from the backends: an email API and a WhatsApp/SMS API.
- The backends contain their own scheduled (cron) jobs that must run on a fixed timezone.

Requirements:
- EACH app folder (both frontends and both backends — four folders total) has its OWN .env
  file. Each container must load only its own .env at runtime (env_file).
- NO secrets may be baked into any image. .env must be blocked via .dockerignore and
  injected at runtime. Images must contain code only.
- Run 24/7: every service restarts automatically on crash and after a host reboot.
- Data must persist across restarts (named volumes for MySQL and Redis).
- Hardening: non-root containers, no-new-privileges, drop all Linux capabilities (add back
  only the minimum MySQL and Redis need to start), read-only root filesystem with tmpfs,
  per-container memory and PID limits, health checks on every service.
- MySQL bound to localhost only; Redis not exposed publicly; both password-protected.
- Put MySQL and Redis on an internal-only Docker network; give the backends a second network
  for outbound internet (email/WhatsApp APIs).
- Node.js heap: set the build-time memory limit inline on the frontend build command
  (--max-old-space-size=1536) and a smaller runtime limit per app.
- The frontend's public API URL is a build-time value (Next.js NEXT_PUBLIC_), so pass it as
  a Docker build arg and bake it at build.

Give me: every Dockerfile, the docker-compose.yml, the four .env templates, the .dockerignore,
the MySQL init SQL (create both databases + a shared app user with access to both), and clear
run instructions. Keep it generic — do not assume any specific company or product name.
```

---

## B. Run steps

### 1. Put files and code in place
Create the folder layout and copy in your real code (keep each Dockerfile + .dockerignore):
```
stack/
├── docker-compose.yml
├── db/init/00-init.sql
├── application-1/backend/   (Dockerfile, .dockerignore, .env, your code)
├── application-1/frontend/  (Dockerfile, .dockerignore, .env, your code)
├── application-2/backend/   (Dockerfile, .dockerignore, .env, your code)
└── application-2/frontend/  (Dockerfile, .dockerignore, .env, your code)
```

### 2. Fill the four `.env` files
Edit each app folder's `.env` (templates in the files document). Generate strong secrets:
```bash
openssl rand -base64 32     # or:  openssl rand -hex 24   (no special chars — safer)
```
Lock them down:
```bash
chmod 600 application-1/backend/.env application-1/frontend/.env \
          application-2/backend/.env application-2/frontend/.env
```

> **Using the team's real `.env` (often with MANY variables)?** Don't rewrite it. Drop it in
> as-is and change only the host variables. See **Section F** for the exact method.

### 3. Create the small root `.env` (shared infra values)
Beside `docker-compose.yml`, create a root `.env` for the values MySQL/Redis/compose need.
These MUST match the DB/Redis passwords inside the app `.env` files:
```bash
cat > .env << 'EOF'
MYSQL_ROOT_PASSWORD=change_me_root
MYSQL_APP_PASSWORD=change_me_app_password
REDIS_PASSWORD=change_me_redis_password
# Public URL the BROWSER uses to reach each backend (baked into the frontends).
# Local/tunnel: http://localhost:5001  |  Public: http://<server-ip>:5001
APP1_PUBLIC_API_BASE_URL=http://localhost:5001
APP2_PUBLIC_API_BASE_URL=http://localhost:5002
EOF
chmod 600 .env
```
> Use `hex` passwords (`openssl rand -hex 24`) to avoid special characters that can break
> the Redis/MySQL startup.

### 4. Make Docker start on boot (reboot persistence)
```bash
sudo systemctl enable docker
```

### 5. Build and start (24/7)
```bash
docker compose up -d --build
docker compose ps          # wait until mysql + redis are healthy
```

### 6. Load your real data into the LOCAL database
The `mysql:` container **is** your local database — the init SQL already created both empty
databases (`app1_db`, `app2_db`) on first start, so you only feed data into them. Full
guidance (laptop file, S3, gzipped, streaming, migrations) is in **Section G**. The common
case:
```bash
docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app1_db < app1-dump.sql
docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app2_db < app2-dump.sql
```

### 7. Verify
```bash
curl http://localhost:5001/health      # app1 backend
curl http://localhost:5002/health      # app2 backend
curl -I http://localhost:3001          # app1 frontend
curl -I http://localhost:3002          # app2 frontend
```

### 8. View in a browser

**Option A — SSH tunnel (recommended, secure, no rebuild, no firewall change):**
```
ssh -i "key.pem" -L 3001:localhost:3001 -L 5001:localhost:5001 -L 3002:localhost:3002 -L 5002:localhost:5002 <user>@<server-ip>
```
Then browse to `http://localhost:3001` and `http://localhost:3002` on your laptop.
(With the tunnel, the frontend's `localhost:5001` API URL works as-is — no rebuild needed.)

**Option B — Public access (less secure; only for a quick demo):**
1. In the root `.env`, set the public URLs to the server IP:
   ```
   APP1_PUBLIC_API_BASE_URL=http://<server-ip>:5001
   APP2_PUBLIC_API_BASE_URL=http://<server-ip>:5002
   ```
2. Change the port bindings in `docker-compose.yml` from `127.0.0.1:3001:3001` to
   `3001:3001` (and the same for 3002/5001/5002).
3. Rebuild the frontends so the new URL is baked in:
   ```bash
   docker compose up -d --build app1-frontend app2-frontend
   ```
4. Open ports 3001/3002/5001/5002 in the cloud firewall (restrict source to your IP).
5. Browse to `http://<server-ip>:3001`. **Revert these changes after the demo.**

### 9. Prove 24/7 persistence
```bash
docker compose kill app1-backend
docker compose ps        # down → wait ~10s → restarted itself
sudo reboot              # reconnect later → docker compose ps → all back up
```

---

## C. Troubleshooting (every issue, with the fix)

### 1. Build fails: `no space left on device`
The disk filled with old images/build cache (common after several rebuilds on a small disk).
```bash
docker system prune -af        # removes unused images + build cache (keeps volumes/data)
df -h /                         # check free space
docker system df                # see what Docker is using
```
Then rebuild with a **normal** build (avoid `--no-cache`, which doubles disk use):
```bash
docker compose up -d --build
```
If the root disk is small (e.g. 8 GB), grow the EBS volume — rebuilds will keep filling it.

### 2. Redis unhealthy: `setpriv: setresuid failed: Operation not permitted`
`cap_drop: ALL` removed the capability Redis needs to drop to its user. Add them back:
```yaml
  redis:
    cap_drop: [ALL]
    cap_add: [SETUID, SETGID, SETPCAP, CHOWN, DAC_OVERRIDE]
```
Then `docker compose up -d`.

### 3. DB error: `Access denied for user 'app_user'@'...' (using password: YES)`
The MySQL data volume was created earlier with a different password, OR the init SQL's
`${VAR}` didn't expand (the `.sql` file does NOT expand shell variables). Fix: let the MySQL
image create the user via env (`MYSQL_USER`/`MYSQL_PASSWORD` in compose), keep only the
second-DB GRANT in the SQL, then reset the volume (safe only if no real data yet):
```bash
docker compose down -v       # wipes MySQL volume so init re-runs with correct password
docker compose up -d
```

### 4. Browser: `This site can't be reached` / `ERR_CONNECTION_TIMED_OUT`
Timeout = the cloud firewall is blocking the port, OR the container is bound to localhost.
- Check binding: `docker compose ps` — must show `0.0.0.0:3001->3001/tcp` (not `127.0.0.1`).
- Check the security group has an inbound rule for that port.
- Prefer the **SSH tunnel** (Option A) — no firewall change needed.

### 5. Browser: `This site can't be reached` on `https://`
The app serves **http**, not https (no TLS yet). Use `http://...`, and try an Incognito
window (browsers cache/force https). HTTPS comes later via a reverse proxy / load balancer.

### 6. Page loads but shows `{"error":"Failed to fetch"}` on login
The frontend is calling the backend at a URL the browser can't reach (usually `localhost`
when viewing from another machine). Fix the public API URL and rebuild the frontends:
```bash
# set APP1_PUBLIC_API_BASE_URL / APP2_PUBLIC_API_BASE_URL to the reachable URL, then:
docker compose up -d --build app1-frontend app2-frontend
```
Then hard-refresh (Ctrl+Shift+R) or use Incognito. Confirm in F12 → Network which URL the
failed request used. (With the SSH tunnel, keep `localhost` and no rebuild is needed.)

### 7. Backend keeps restarting
Wrong start command or DB not ready.
```bash
docker compose logs --tail 40 app1-backend
```
- No `start` script → add `"start": "node src/index.js"` (your entry file) to package.json.
- Health route isn't `/health` → edit the `HEALTHCHECK` path in that Dockerfile.

### 8. `read-only file system` error
The app writes to disk (uploads/logs). Either remove `read_only: true` for that service, or
mount a writable volume for the specific path it writes to.

### 9. Native module build fails (e.g. bcrypt, swc)
Alpine lacks build tools. Add to the build stage of that Dockerfile:
```dockerfile
RUN apk add --no-cache python3 make g++
```

### 10. Cron job runs twice
The scheduler lives inside the backend and the backend was scaled to more than one copy.
Keep that backend at **one replica** while the cron is inside it (split the scheduler into a
single dedicated worker before scaling the web tier).

### 11. `port is already allocated`
Another process uses the port. Stop it, or change the host port in `docker-compose.yml`.

### 12. Changed a backend `.env` but the container didn't pick it up
`env_file` is read at container start. Recreate the container:
```bash
docker compose up -d app1-backend     # recreates with the new .env
```
(Frontend `NEXT_PUBLIC_*` values need a **rebuild**, not just a restart — see item 6.)

### 13. Dump loaded into the wrong database / app can't find tables
The database name in the dump didn't match the app's `DB_NAME`. Confirm names:
```bash
head -40 your-dump.sql                  # look for any CREATE DATABASE / USE line
docker compose exec mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "SHOW DATABASES;"
```
Either rename the database in the dump, or set the app's `DB_NAME` to match the dump's name —
the two must agree. Then re-load into the correct database.

### 14. `aws: Unable to locate credentials` when pulling the dump from S3
The EC2 has no S3 permission. Either attach an IAM role with read access to the bucket, or
have the team grant bucket access. Confirm with:
```bash
aws sts get-caller-identity     # returns an identity once the role/access is in place
```

---

## D. Everyday commands
```bash
docker compose ps                       # status
docker compose logs -f app1-backend     # follow logs (check cron: look for "scheduler")
docker compose restart app2-backend     # restart one service
docker compose down                     # stop all (data persists in volumes)
docker compose up -d --build            # rebuild + start after code change
docker system prune -af                 # free disk (keeps volumes)
```

## E. Security reminders
- Each `.env` is `chmod 600` and gitignored; never commit it; never `cat` it on a shared screen.
- Secrets are backend-side only; the frontend gets a URL, never a secret.
- If you opened public ports for a demo, **revert** to `127.0.0.1` bindings and remove the
  firewall rules afterward.
- For real production access, use HTTPS via a reverse proxy / load balancer — not raw public
  ports.

---

## F. Integrating the team's real `.env` (when it has MANY variables)

When the team hands you a `.env` with dozens of variables, **do not rewrite it and do not try
to understand every line.** `env_file:` loads the whole file automatically. You only change
the few variables that point at a host, because inside Docker, services talk by **service
name**, not `localhost`.

**Step 1 — Use their file as-is.** Drop their real `.env` into the matching app folder. The
compose already loads all of it via `env_file:`.

**Step 2 — Change ONLY the host variables:**

| Their `.env` likely has           | Change it to       | Why                                       |
|-----------------------------------|--------------------|-------------------------------------------|
| `DB_HOST=localhost` / `127.0.0.1` | `DB_HOST=mysql`    | MySQL is reached by service name in Docker|
| `REDIS_HOST=localhost`            | `REDIS_HOST=redis` | same — Redis by service name              |
| `DB_PORT=3306`                    | *leave as-is*      | port is unchanged                         |

Everything else — JWT secrets, email/WhatsApp keys, cookie names, feature flags, all the
other variables — **leave exactly as they are.**

**Step 3 — Find what to change (don't read all of them):**
```bash
grep -iE "host|port|redis|mysql|database|localhost|127.0.0.1" application-1/backend/.env
```
Only the lines pointing at `localhost` need editing — usually 2–3.

**Step 4 — Keep their variable NAMES unchanged.** Their code reads specific names
(e.g. `process.env.DB_HOST`). Keep the names; change only the host *value*. Nothing else
needs renaming for the code to find its variables.

**Step 5 — Frontend `.env`:** only `NEXT_PUBLIC_API_BASE_URL` matters (the URL the browser
uses). Leave the rest. Because `NEXT_PUBLIC_*` is baked at build, changing it needs a frontend
**rebuild**, not a restart.

**Step 6 — Apply:**
```bash
chmod 600 application-1/backend/.env
docker compose up -d app1-backend     # recreate so it re-reads the .env
```

---

## G. Loading data into the LOCAL database (full guide)

"Local database" = the `mysql:` container in this stack. Its data lives in the `mysql_data`
volume on the server. The init SQL already created the two empty databases on first start, so
**you never create databases by hand — you only feed data in.**

### Step 1 — Inspect the dump first
```bash
head -40 your-dump.sql
```
- **Case A — has `CREATE DATABASE` / `USE`:** it builds the DB itself; just load it.
- **Case B — only `CREATE TABLE ...`:** loads into the DB you name in the command (your
  databases already exist, so this works).

> ⚠️ If the dump's `CREATE DATABASE` name differs from `app1_db`/`app2_db`, either rename it
> in the dump or set the app's `DB_NAME` to match. The dump name and the app `.env` name must
> agree.

### Source A — dump file on your laptop
```bash
scp -i "key.pem" app1-dump.sql ubuntu@<server-ip>:~/stack/
scp -i "key.pem" app2-dump.sql ubuntu@<server-ip>:~/stack/
docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app1_db < app1-dump.sql
docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app2_db < app2-dump.sql
```

### Source B — dump from S3
```bash
aws sts get-caller-identity                                  # confirm the EC2 can reach S3
aws s3 cp s3://your-bucket/path/app1-dump.sql ./app1-dump.sql
docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app1_db < app1-dump.sql
```
**Gzipped, no disk write:**
```bash
gunzip -c app1-dump.sql.gz | docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app1_db
```
**Stream straight from S3 into MySQL (never touches the server disk):**
```bash
aws s3 cp s3://your-bucket/path/app1-dump.sql - | docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app1_db
aws s3 cp s3://your-bucket/path/app1-dump.sql.gz - | gunzip -c | docker compose exec -T mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" app1_db
```

### Source C — app builds its own tables (migrations)
If the backend runs migrations on startup, the tables build themselves when it starts —
nothing to load manually. Just bring the stack up and let it migrate.

### Verify
```bash
docker compose exec mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "USE app1_db; SHOW TABLES;"
docker compose exec mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "USE app2_db; SHOW TABLES;"
```
You should see your tables under each database.

> If `$MYSQL_ROOT_PASSWORD` isn't set in your shell, type the real password in the command:
> `... -p"YourActualRootPassword" app1_db < app1-dump.sql`
