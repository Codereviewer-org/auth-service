# Auth Service

FastAPI-based authentication service for Code Reviewer. It provides user registration, login, health checks, readiness checks, and simple Prometheus-style request metrics. User records are stored in PostgreSQL.

## Tech Stack

- Python 3.10
- FastAPI
- Uvicorn
- PostgreSQL
- psycopg2
- Docker
- GitHub Actions reusable CI/CD workflows

## Project Structure

```text
.
├── .github/workflows/ci-cd.yaml
├── Dockerfile
├── main.py
├── README.md
└── requirements.txt
```

## Configuration

The service reads database configuration from `DATABASE_URL`.

If `DATABASE_URL` is not set, the service uses this default:

```text
postgresql://coderaptor:coderaptor@postgres:5432/coderaptor
```

The app also supports secret-file based configuration. If `DATABASE_URL` is missing, it checks:

```text
DATABASE_URL_FILE
/mnt/secrets-store/DATABASE_URL
```

Optional environment variables:

| Variable | Description | Default |
| --- | --- | --- |
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://coderaptor:coderaptor@postgres:5432/coderaptor` |
| `DATABASE_URL_FILE` | Path to a file containing the database URL | `/mnt/secrets-store/DATABASE_URL` |
| `LOG_LEVEL` | Python logging level | `INFO` |

## Local Setup

Create and activate a virtual environment:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

Install dependencies:

```powershell
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

Set the database URL:

```powershell
$env:DATABASE_URL = "postgresql://coderaptor:coderaptor@localhost:5432/coderaptor"
```

Run the service:

```powershell
uvicorn main:app --host 0.0.0.0 --port 8001 --reload
```

Open the API docs:

```text
http://localhost:8001/docs
```

## API Endpoints

| Method | Path | Description |
| --- | --- | --- |
| `GET` | `/health` | Basic health check |
| `GET` | `/live` | Liveness check |
| `GET` | `/ready` | Readiness check that verifies database connectivity |
| `GET` | `/metrics` | Prometheus-style request metrics |
| `POST` | `/register` | Register a new user |
| `POST` | `/login` | Login with email and password |

### Register

```powershell
Invoke-RestMethod `
  -Method Post `
  -Uri "http://localhost:8001/register" `
  -ContentType "application/json" `
  -Body '{"username":"demo","email":"demo@example.com","password":"demo123"}'
```

### Login

```powershell
Invoke-RestMethod `
  -Method Post `
  -Uri "http://localhost:8001/login" `
  -ContentType "application/json" `
  -Body '{"email":"demo@example.com","password":"demo123"}'
```

Successful login response:

```json
{
  "message": "Login successful",
  "username": "demo",
  "email": "demo@example.com"
}
```

## Database

On startup, the service creates the `users` table if it does not already exist:

```sql
CREATE TABLE IF NOT EXISTS users (
  username TEXT PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  password TEXT NOT NULL
);
```

Passwords are currently hashed with SHA-256 and a static salt in `main.py`. For production hardening, replace this with a password hashing algorithm such as bcrypt or Argon2 and return signed tokens from `/login`.

## Docker

Build the image:

```powershell
docker build -t auth-service:local .
```

Run the container:

```powershell
docker run --rm -p 8001:8001 `
  -e DATABASE_URL="postgresql://coderaptor:coderaptor@host.docker.internal:5432/coderaptor" `
  auth-service:local
```

The Dockerfile uses a multi-stage Python build and runs the application as a non-root user.

## CI/CD

The workflow in `.github/workflows/ci-cd.yaml` uses shared workflows from:

```text
CodeReviewer-org/codereviewer-main
```

For pushes and pull requests to `main`, it runs CI and updates the dev deployment metadata for:

```text
auth-service
```

Manual production releases are started with `workflow_dispatch` and require a `release_tag`.

Required GitHub secrets include:

- `ACR_LOGIN_SERVER`
- `ACR_USERNAME`
- `ACR_PASSWORD`
- `INFRA_REPO_TOKEN`
- `EMAIL_USERNAME`
- `EMAIL_PASSWORD`

CI also references these optional security and quality secrets:

- `SONAR_TOKEN`
- `SNYK_TOKEN`

## Validation

Run linting:

```powershell
ruff check .
```

Run tests:

```powershell
pytest -q
```

At the moment, this repository does not include a committed test suite. Add tests before relying on `pytest -q` as a deployment gate.
