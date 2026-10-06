# Ship It - Two Tone DevOps / Cloud Challenge

A containerised Flask API (`/` and `/health`) that runs together with PostgreSQL using Docker Compose, with a GitHub Actions pipeline that tests the app and builds the image on every push.

**Repo:** https://github.com/Refiloe07/devOps_cloud_take_home_challenge

## Run it locally (one command)

Requires Docker with the Compose plugin.

```bash
cp .env.example .env && docker compose up --build
```

On Windows PowerShell: `Copy-Item .env.example .env; docker compose up --build`

Then open <http://localhost:8000/health>. Expected:

```json
{"database": "connected", "status": "ok"}
```

Before the first run you can edit `POSTGRES_PASSWORD` in `.env` (letters and numbers only, because it is embedded in a connection URL). Stop with `docker compose down`; add `-v` to also delete the database volume.

Run the tests without Docker:

```bash
cd starter-app
pip install -r requirements-dev.txt
pytest
```

## Deliverables checklist

- [x] 1. Dockerfile: multi-stage, slim base, non-root user
- [x] 2. docker-compose.yml: app + PostgreSQL; `/health` reports `"database": "connected"`
- [x] 3. DB password from `.env` (git-ignored); only `.env.example` is committed
- [x] 4. CI pipeline: on push, install deps, run tests, build image
- [ ] 5. Pipeline proven to fail on a broken test (red commit, then fix): see the Actions history
- [x] 6. Health check using `/health` (compose healthcheck + CI smoke test)
- [x] README: one-command run + deployment plan

## How it works

- **Dockerfile**: Stage 1 creates a virtualenv and installs `requirements.txt`. Stage 2 starts from a clean `python:3.12-slim`, copies only the virtualenv and `app.py`, and runs gunicorn as a non-root user. Build tools, pip cache and tests never reach the final image (about 236 MB on disk, about 57 MB compressed).
- **`.dockerignore`**: keeps `.git`, `.env`, caches, tests and docs out of the build context.
- **docker-compose.yml**:
  - `db` (PostgreSQL 16 alpine) has a `pg_isready` health check and stores data in the `db-data` volume.
  - `app` waits for the database to be healthy (`depends_on: service_healthy`), and has its own health check that calls `/health`. It uses Python rather than `curl`, which the slim image doesn't include.
  - Both services use `restart: unless-stopped`.
- **Secrets**: the DB password is read from `.env`, which is git-ignored. Compose refuses to start if `POSTGRES_PASSWORD` is missing. `.env.example` documents the variables with a placeholder value.
- **CI** (`.github/workflows/ci.yml`):
  - `test` job: installs `requirements-dev.txt` and runs `pytest`.
  - `build` job (`needs: test`, so it is skipped if tests fail): builds the Docker image, then starts the full stack with a random throwaway password generated for that run and checks that `/health` reports `connected`.
  - No password is stored in the repo or the pipeline file.
- **Requirements split**: `requirements.txt` is what runs in production; `requirements-dev.txt` adds pytest for CI and local testing only.

## Deployment plan (AWS)

**Services**
- **ECR** stores the image. **ECS on Fargate** runs it (no servers to patch) behind an **Application Load Balancer**, whose target-group health check calls `/health`.
- **RDS for PostgreSQL** in private subnets, reachable only from the app's security group. Multi-AZ in production.
- **Route 53 + ACM** for DNS and HTTPS.
- Infrastructure is defined in **Terraform** so staging and production are built from the same code.

**Staging vs production**
- Two separate environments (ideally two AWS accounts) with identical infrastructure, differing only in variables: instance sizes, Multi-AZ, domain name.
- On every push, CI tests and builds. Merging to `main` pushes the image to ECR tagged with the commit SHA and deploys it automatically to **staging**. Production needs a manual approval (GitHub Environments) and deploys the **same image** that was tested: build once, promote.
- GitHub authenticates to AWS with OIDC and short-lived roles, so no long-lived AWS keys are stored in GitHub.

**Secrets**
- Database credentials live in **AWS Secrets Manager** (RDS can rotate them) and are injected into the ECS task as environment variables at start-up.
- Task roles follow least privilege. Nothing secret is in the image, the repo or the pipeline.

**Rollback**
- Images are immutable and tagged by commit SHA, so rolling back means redeploying the previous ECS task definition revision.
- ECS rolling deployments with the **deployment circuit breaker** automatically roll back if new tasks fail the load balancer health check, so a bad release never takes over fully.
- Database changes are made backwards-compatible (add before remove) so the previous version still works after a rollback. RDS automated backups and snapshots cover data recovery.

**Monitoring**
- **CloudWatch Logs** collects container output.
- **CloudWatch alarms** on: ALB 5xx rate, unhealthy targets, task CPU/memory, RDS CPU, connections and free storage. Alarms notify the team through **SNS** (email or Slack).
- A synthetic check on `/health` from outside AWS to catch outages the alarms miss. Tracing (OpenTelemetry/X-Ray) can be added as the app grows.

## Assumptions

- Docker with Compose v2 (`docker compose`) is installed.
- The app is unchanged, as the brief says; only packaging and automation were added.
- `pytest` is kept out of the production image via a separate `requirements-dev.txt`.
- DB user and name default to `twotone`; the password has no default on purpose.
- The database only needs `SELECT 1`, so there are no migrations.
- CI runs on every push (and on pull requests); the brief only requires push.
- The deployment plan is a design only: nothing is deployed, since no cloud account is needed.
