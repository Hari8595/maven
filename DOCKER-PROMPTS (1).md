# Docker Setup Prompts — Two Applications

Two ready-to-use prompts for ChatGPT (or any AI). Copy the one you need.
Both explain the architecture in simple language and include the fixes learned from real
deployment, so the AI won't lead you into the same errors.

---

## Which prompt to use

- **Prompt 1** — MySQL runs on the **host** server (with Apache), Redis in Docker.
  *(This matches the real current setup.)*
- **Prompt 2** — **Both** MySQL and Redis run in Docker (fully self-contained stack).

---

## The architecture (plain summary)

- **Two web applications.** Each has a **Next.js frontend** + a **Node.js 22 backend** → 4 app folders.
- **One MySQL, two databases** (one per app). **Both backends connect to both databases**
  (one app promotes records into the other; shared staff use both).
- **Sequelize ORM with sync OFF** → the app does **not** auto-create tables. Tables + data come
  from a **SQL dump** loaded once.
- **Redis** = message queue (fills itself, no data setup).
- **Cron jobs** run inside the backends (repeating scheduled loops).
- **Four separate `.env` files** (one per app folder). Many variables each. **No secrets baked
  into images** — `.env` loaded at runtime only.

---

## PROMPT 1 — MySQL on the HOST + Redis in Docker

```
Act as a DevOps expert and explain simply, step by step — I am a junior engineer.

MY ARCHITECTURE:
- TWO web applications. Each has a Next.js frontend and a Node.js 22 backend (so 4 app folders total).
- MySQL is installed on the HOST server (same machine, alongside Apache). It is NOT in Docker.
- There are two databases (one per application). Both backends connect to BOTH databases.
- The backends use Sequelize ORM with "sync" turned OFF, so the app does NOT auto-create tables. The tables and data come from a SQL DUMP that is loaded into the host MySQL once.
- Redis runs as a Docker CONTAINER, used as a message queue. It needs no data setup.
- The backends contain their own cron jobs (repeating scheduled loops).
- Each app folder has its OWN .env file (4 total). The .env files have many variables (DB, Redis, JWT, email API, WhatsApp API). Secrets must NOT be baked into images — load .env at runtime only.

WHAT I NEED:
1. A docker-compose.yml that runs: both backends, both frontends, and Redis — but NOT MySQL (MySQL is on the host).
2. The backend containers must connect to MySQL running on the HOST. Explain clearly how a container reaches host MySQL — use host.docker.internal with extra_hosts: ["host.docker.internal:host-gateway"] (since localhost inside a container means the container itself, not the host). I want the DB host value OVERRIDDEN to point to the host, while keeping the rest of each app's .env unchanged.
3. Each container must load its own .env file using env_file.
4. Dockerfiles for Node.js backends and Next.js frontends (non-root, no secrets in image).
5. Must run 24/7 and auto-restart on crash or server reboot.
6. Show me how to LOAD THE SQL DUMP into the host MySQL (the command), and how to confirm the host MySQL accepts connections from the Docker containers (bind-address and user grants for the Docker IP range).

IMPORTANT:
- For Next.js, the public API URL (NEXT_PUBLIC_) is set at BUILD time, so pass it as a Docker build arg.
- The frontend build needs more memory: use NODE_OPTIONS="--max-old-space-size=1536" on the build command.
- Keep MySQL OUT of docker-compose; keep Redis IN docker-compose.

Please give me the files one at a time, with comments, in simple language. Also tell me which .env variables I must change for Docker (the DB host and Redis host) and which to leave exactly as they are.
```

---

## PROMPT 2 — Recreate BOTH MySQL and Redis in Docker

```
Act as a DevOps expert and explain simply, step by step — I am a junior engineer.

MY ARCHITECTURE:
- TWO web applications. Each has a Next.js frontend and a Node.js 22 backend (4 app folders total).
- I want BOTH MySQL and Redis to run as Docker CONTAINERS inside my docker-compose (a fully self-contained stack — nothing on the host).
- One MySQL container hosting TWO databases (one per application). Both backends connect to BOTH databases.
- The backends use Sequelize ORM with "sync" turned OFF, so the app does NOT auto-create tables. The tables and data come from a SQL DUMP that I will load into the MySQL container.
- Redis container is used as a message queue. It needs no data setup.
- The backends contain their own cron jobs (repeating scheduled loops).
- Each app folder has its OWN .env file (4 total) with many variables (DB, Redis, JWT, email API, WhatsApp API). Secrets must NOT be baked into images — load .env at runtime only.

WHAT I NEED:
1. A docker-compose.yml that runs everything: MySQL, Redis, both backends, both frontends.
2. Inside Docker, containers reach each other by SERVICE NAME. So the backends should use DB_HOST=mysql and REDIS_HOST=redis (NOT localhost, NOT an IP). I want these host values OVERRIDDEN to the service names, while keeping the rest of each app's .env unchanged.
3. An init SQL that creates BOTH databases and one app user with access to BOTH databases. (Note: the MySQL official image expands env variables to create the user; a plain .sql file does NOT expand ${VARIABLES} — so create the user via the MySQL image's environment, and use the init SQL only for the second database grant.)
4. How to LOAD THE SQL DUMP into the MySQL container after it starts (the exact command), since Sequelize sync is off and the tables must come from the dump.
5. Persistent data using named volumes (MySQL data and Redis data survive restarts and rebuilds).
6. Each container loads its own .env using env_file.
7. Dockerfiles for Node.js backends and Next.js frontends (non-root, no secrets in image).
8. Must run 24/7 and auto-restart on crash or reboot.

SECURITY / HARDENING (please include):
- Non-root containers, no-new-privileges, drop all Linux capabilities and add back only the minimum MySQL and Redis need to start (Redis needs SETUID, SETGID, SETPCAP, CHOWN, DAC_OVERRIDE or it fails with "setpriv: setresuid failed").
- MySQL bound to localhost only on the host; Redis not exposed publicly; both password-protected.
- Use passwords with NO special characters (generate with: openssl rand -hex 24) to avoid MySQL/Redis startup errors.

IMPORTANT:
- For Next.js, the public API URL (NEXT_PUBLIC_) is set at BUILD time, so pass it as a Docker build arg.
- The frontend build needs more memory: use NODE_OPTIONS="--max-old-space-size=1536" on the build command.

Please give me the files one at a time, with comments, in simple language. Also tell me exactly how to load the SQL dump and how to verify the tables loaded.
```

---

## Tips for using these with ChatGPT (free version)

1. **The AI can't see your files** — it only knows what you paste. When it needs your `.env`
   variables or folder names, paste them (mask secret values with `xxx`).
2. **Shorter memory on free version** — in a long chat it may forget. If it drifts,
   **re-paste the architecture block** to bring it back.
3. **Ask for one file at a time** — you get better quality than "give me everything at once".
4. **The most common mistake to watch for:** if the AI says to use `localhost` for the DB host
   in Prompt 1, that's WRONG — a container's localhost is itself. Insist on
   `host.docker.internal` (with `extra_hosts`).

---

## The one thing to confirm with your team (both prompts)

Because Sequelize **sync is OFF**, the tables can only come from the **SQL dump**.
So confirm the dump contains the schema:

```
grep -c "CREATE TABLE" your-dump.sql
```

If the result is greater than 0 → the schema is in the dump (good — load it once).
If it is 0 → the dump is data-only; ask the team for a dump that includes the table structure,
or for the schema/migration SQL separately. Without it, the app will start but error when it
queries a table that does not exist.
```
