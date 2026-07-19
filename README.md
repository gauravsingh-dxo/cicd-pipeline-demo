# cicd-pipeline-demo

A complete, runnable CI/CD pipeline — not just a YAML file, but an actual small API with tests, wired through a pipeline that tests, builds, pushes, and gates production deploys behind manual approval.

## What this demonstrates

- Tests run *before* anything gets built — a failed test means no image is ever pushed
- Multi-stage Dockerfile that keeps devDependencies and source maps out of the final image
- Image tagging with the git SHA (not just `latest`), so any previous build can be redeployed on demand
- GitHub Actions cache (`type=gha`) for Docker layer caching — meaningfully faster rebuilds on small changes
- A production deploy job gated behind a GitHub Environment, which requires a human to click approve before it runs
- Container runs as a non-root `USER node`, with a `HEALTHCHECK` baked into the image itself

## Architecture

```
push to main
     │
     ▼
  ┌──────┐     ┌────────────────┐     ┌──────────────────┐
  │ test │ ──▶ │ build-and-push │ ──▶ │ deploy (gated by  │
  │      │     │  (tag: SHA +   │     │  manual approval  │
  │      │     │   latest)      │     │  on 'production') │
  └──────┘     └────────────────┘     └──────────────────┘
```

## Prerequisites

- Node.js 20+ (for running locally)
- Docker (for building the image locally)
- A GitHub repo with a `production` Environment configured under Settings → Environments, with at least one required reviewer, to see the approval gate in action

## Usage

**Run locally:**
```bash
npm install
npm test
npm start
# curl http://localhost:3000/health
```

**Run in Docker:**
```bash
docker build -t devops-portfolio-api .
docker run -p 3000:3000 devops-portfolio-api
```

**Trigger the pipeline:** push to `main` (or open a PR to see just the test job run).

## Design decisions

**Why GitHub Container Registry instead of AWS ECR?** This is meant to be forked and run by anyone reviewing it, without needing AWS credentials configured first. `GITHUB_TOKEN` auth to GHCR works out of the box on any GitHub repo. In an actual production setup I'd point this at ECR/ACR/GCR using OIDC federation (no long-lived cloud keys stored as GitHub secrets) — happy to show that version too, it's a small swap in the `build-and-push` job.

**Why tag with both the SHA and `latest`?** `latest` is a convenience pointer for "what's currently on main," but it should never be what you deploy from directly — the SHA tag is what the deploy job actually references, so rollback is just redeploying a known-good SHA, not rebuilding from a git revert under pressure.

**Why gate deploy behind a GitHub Environment instead of a branch protection rule?** Environments give you a real approval UI, deployment history, and the ability to restrict which secrets are even visible to that job — branch protection only controls what can merge, not what can deploy.
