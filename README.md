# Ship It — Two Tone DevOps / Cloud Challenge

TODO: one-line description of the app.

## Run it locally (one command)

TODO: the single command that starts the app + database.

## Deliverables checklist

- [ ] 1. Dockerfile — multi-stage, small image
- [ ] 2. docker-compose.yml — app + PostgreSQL; `/health` reports `"database": "connected"`
- [ ] 3. DB password from env var / `.env`, not committed
- [ ] 4. CI pipeline — on push: install deps, run tests, build image
- [ ] 5. Prove the pipeline fails on a broken test (red commit, then fix)
- [ ] 6. Health check using the `/health` endpoint
- [ ] README: one-command run + one-page AWS/Azure deployment plan

## Deployment plan (AWS or Azure)

TODO: name the services you'd use, and cover staging vs production, secrets,
rollback, and monitoring.

## Assumptions

TODO: note any sensible assumptions you made (the brief rewards this).
