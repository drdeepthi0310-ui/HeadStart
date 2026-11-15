# HEADSTART — one command, Windows

Whole app (FastAPI backend + React frontend + AI analyst) in one Docker image.
No Python, no Node, no npm. Only **Docker Desktop** is required.

---

## Run it

1. Extract `HeadStart.zip` anywhere (Desktop is fine).
2. Double-click **`start-headstart.bat`**.

That is the whole procedure. The first run loads the image (~1 minute), starts
the container, waits for the API, and opens **http://localhost:8000** in your
browser. Later runs start in a few seconds.

Same thing from a terminal, if you prefer:

```bat
start-headstart.bat
```

Port 8000 already taken? Pass another port:

```bat
start-headstart.bat 8080
```

Then open **http://localhost:8080**.

**Docker Desktop must be running** — start it and wait until it says
"Engine running". The script tells you if it is not.

---

## Create your login

There is no pre-made account in a fresh install. On the sign-in screen choose
**Create an account** — email plus a password of **at least 12 characters**.
Data is stored in the `headstart-data` Docker volume and survives restarts.

Then try: **Markets**, **Assets** (search `AAPL`), **Portfolio** (add a
transaction), **Alerts**, and **AI Analyst** (ask *"compare AAPL and MSFT"* or
*"how is the market today?"*).

---

## What is inside

| File | Purpose |
| --- | --- |
| `headstart-1.0.0.tar.gz` | The application image (backend + built frontend) |
| `start-headstart.bat` | One-command launcher (loads image, runs container, opens browser) |
| `docker-compose.windows.yml` | Optional PostgreSQL setup instead of SQLite |
| `IMAGE-INFO.txt` | Image id, checksum, build details |
| `previews/` | Screenshots of every main screen |
| `README-WINDOWS.md` | This file |

---

## Managing it

```bat
docker logs -f headstart      :: live logs
docker stop headstart         :: stop
docker start headstart        :: start again (data kept)
docker rm -f headstart        :: remove the container
docker volume rm headstart-data   :: delete the database too
```

Optional PostgreSQL instead of SQLite:

```bat
docker compose -f docker-compose.windows.yml up -d
docker compose -f docker-compose.windows.yml down
```

---

## Notes

- **AI Analyst** works out of the box: the Groq key is compiled into the app
  (`backend/core/config.py`) and also passed by the launcher. Set the
  `GROQ_API_KEY` environment variable to override it. Without any key the API
  returns a clean "provider not configured" error and never fabricates an answer.
- **This package contains a live Groq API key.** Keep `HeadStart.zip` private —
  do not email it, upload it, or commit it. Rotate the key at
  https://console.groq.com/keys when you are done demoing.
- The key is on Groq's free tier: roughly 8,000 tokens/minute, which is about
  6–10 questions per minute. Beyond that you get a clear rate-limit message.
- **Market data** needs outbound HTTPS from the container to Yahoo Finance. If
  quotes show "Unavailable", that network path is blocked.
- **Alert email** needs SMTP settings; without them the app reports
  "Email is not configured" instead of pretending it sent something.
- Container runs as a non-root user (uid 10001) and has a `/health` check.
- Ports: the container always listens on 8000 internally; `start-headstart.bat`
  maps whatever host port you pass onto it. If 8000 is busy it automatically
  picks the next free port (8001, 8002, ...) and opens that.
- The app talks to its API on the same address you opened it from, so
  `localhost`, `127.0.0.1` and any port all work.
- If Docker Desktop is not running, the launcher starts it and waits for the
  engine before continuing.

---

## If something goes wrong

| Symptom | Fix |
| --- | --- |
| "Docker Desktop is not running" | Start Docker Desktop, wait for "Engine running" |
| "port 8000 is already in use" | `start-headstart.bat 8080` |
| Page loads but is blank | Wait 10s and refresh; check `docker logs headstart` |
| Image fails to load | Re-extract the zip (the `.tar.gz` must sit next to the `.bat`) |
| Old colours / small fonts | Hard refresh: `Ctrl + Shift + R` |
