# MoodMentor — Docker Deployment

Production alternative to the Colab + ngrok flow used during development.

## Prerequisites

- Docker and Docker Compose installed
- A reachable PostgreSQL database (e.g. Neon, RDS, or self-hosted)
- A Gmail account with an app password for sending OTP emails

## 1. Export the notebook's source files

The notebook (`Team_B.ipynb`) is the source of truth during development. Before
building images, run every `%%writefile` cell once (top to bottom) so `db.py`,
`auth.py`, `email_utils.py`, `security.py`, `nlp_pipeline.py`,
`recommendations.py`, `backend.py`, and `app.py` exist as plain files
alongside `requirements.txt`, the two `Dockerfile.*` files, and
`docker-compose.yml`.

## 2. Configure environment variables

```bash
cp .env.example .env
```

Fill in `.env` with your real database, JWT, and SMTP credentials. `.env` is
already excluded via `.dockerignore` and should never be committed.

## 3. Build and run

```bash
docker compose up --build
```

- Backend (FastAPI): http://localhost:8000  (health check at `/health`)
- Frontend (Streamlit): http://localhost:8501

The frontend container talks to the backend over the internal Docker network
at `http://backend:8000` (set via `BACKEND_URL` in `docker-compose.yml`), so
no ports need to be exposed publicly except 8501 if you only want to expose
the UI.

## 4. Initialize the database

The app auto-creates/migrates its tables on backend startup via `init_db()`
(same logic used in the Colab notebook), so no separate migration step is
required — just make sure `DB_HOST`/`DB_USER`/`DB_PASSWORD` in `.env` point to
a database the app user can create tables in.

## 5. Stopping / rebuilding

```bash
docker compose down          # stop containers
docker compose up --build    # rebuild after code changes
docker compose logs -f       # tail logs from both services
```

## 6. CI

`.github/workflows/ci.yml` runs on every push/PR to `main`: it installs
dependencies, compiles every module to catch syntax errors, and builds both
Docker images to catch Dockerfile regressions before merge.

## Notes

- `enableCORS` is left off and `enableXsrfProtection` is left on for
  Streamlit — see the comment in the Colab launch cell for why (ngrok origin
  mismatch, not a security trade-off). Behind a fixed domain in production,
  turn `enableCORS` back on.
- Real access control is the JWT bearer token checked on every `backend.py`
  call, not CORS.
