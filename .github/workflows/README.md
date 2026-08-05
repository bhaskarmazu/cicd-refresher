# cicd-refresher — CI/CD Pipeline

A hands-on CI/CD pipeline for a small Flask app, built on GitHub Actions and AWS. It demonstrates a realistic path from a pushed commit to a running production service: automated tests and linting, container vulnerability scanning, static analysis, an approval-gated production deployment, and automatic rollback if a bad deploy slips through.

## Why this exists

This repo was built as a hands-on learning project to go from "I know Docker and AWS" to "I can build and reason about a real CI/CD pipeline end to end" — including the parts that don't show up in tutorials, like OIDC credential federation, branch protection, environment-scoped secrets, and recovering from a real supply chain security incident encountered along the way (see **Incident note** below).

## Architecture

```
Pull request into main
  │
  ├── test    (pytest)
  └── lint    (flake8)
        │
        └── (required checks — a PR cannot merge unless both pass)

Merge / push to main
  │
  ├── test → lint → build-and-push → scan → deploy-staging → deploy-production
  │
  ├── build-and-push   builds the Docker image, authenticates to AWS via OIDC
  │                    (no stored AWS keys), pushes to ECR tagged with the commit SHA
  │
  ├── scan             pulls the pushed image and scans it with Trivy for
  │                    CRITICAL/HIGH vulnerabilities; hard-fails the pipeline if found
  │
  ├── deploy-staging   deploys automatically to a staging container (port 5000)
  │                    on the shared EC2 instance — no approval required
  │
  └── deploy-production deploys to a separate production container (port 5001)
                        on the same instance. Requires a human to click "Approve"
                        in GitHub (Environment protection rule). If the post-deploy
                        health check fails, the previous known-good image is
                        redeployed automatically before the job reports failure.
```

## Pipeline stages in detail

**test** — runs `pytest` against the Flask app's routes. Required check on every pull request.

**lint** — runs `flake8` against `app.py`. Required check on every pull request.

**build-and-push** — builds the Docker image and pushes it to Amazon ECR, tagged with the triggering commit's SHA. Authenticates to AWS using OpenID Connect (`aws-actions/configure-aws-credentials`) — the workflow assumes a scoped IAM role for the duration of the job and never stores a long-lived AWS access key as a secret. Only runs on an actual push to `main` (not on pull requests), so unmerged branches never produce or deploy an image.

**scan** — pulls the freshly-pushed image and scans it with Trivy, run directly as its official Docker image (`aquasec/trivy`) rather than through the `trivy-action` GitHub Action. This is a deliberate choice, explained below. Exits non-zero (blocking the pipeline) if any CRITICAL or HIGH severity vulnerability is found.

**deploy-staging** — deploys the new image to a `app-staging` container on port 5000 of the shared EC2 instance via AWS Systems Manager `send-command` (no SSH exposure needed). Runs automatically after `build-and-push` and `scan` succeed.

**deploy-production** — deploys to a separate `app-production` container on port 5001 of the same instance. This job is tied to a GitHub **Environment** named `production`, which has a required-reviewer protection rule: the job pauses and waits for a human approval click before it runs at all. After deploying, it runs a health check against the new container; if that check fails, it automatically redeploys the last known-good image tag (tracked in a small state file on the instance) and still reports the job as failed — so a bad deploy is both caught and self-healed before anyone needs to intervene manually.

**CodeQL** — GitHub's built-in static analysis (default setup), running independently of the custom workflow above, scanning for common code-level security issues.

## Security practices

- **No stored AWS credentials.** Every AWS-touching job authenticates via OIDC federation to a scoped IAM role (`github-actions-ecr-push`), trusted only for this specific repository.
- **Branch protection.** A ruleset on `main` requires `test`, `lint`, `scan`, and CodeQL to all pass, and requires every change to go through a reviewed pull request — verified by deliberately breaking a test and confirming the merge button was blocked until it was fixed.
- **Environment-scoped configuration.** `staging` and `production` each have their own GitHub Environment variables (`APP_PORT`, `CONTAINER_NAME`), so the same workflow code deploys to two independent targets without duplicated YAML.
- **Approval gate on production.** Deployments to `production` require a human reviewer to explicitly approve the run — staging deploys automatically, production does not.
- **Automatic rollback.** A failed production health check triggers an automatic redeploy of the last known-good image before the pipeline reports failure.

## Incident note: the trivy-action supply chain attack

While building the `scan` job, the version originally pinned (`aquasecurity/trivy-action@0.28.0`) turned out to reference a tag that never existed. Investigating the correct version surfaced a real, disclosed supply chain attack: in March 2026, attackers compromised `trivy-action`'s GitHub infrastructure and force-pushed the large majority of its version tags to malicious commits designed to exfiltrate cloud credentials from CI runs. The official guidance was that even GitHub's "Immutable release" badge does not prevent a tag from being force-pushed — full commit-SHA pinning was the only reliable protection at the time.

Rather than keep depending on that action's own binary-installation logic (which itself began failing during the incident's cleanup), this pipeline scans images by running Trivy's official Docker image directly (`aquasec/trivy:0.35.0`) — a separate distribution channel from the compromised GitHub Action and its release tags entirely. This is the same reasoning that led the pipeline toward least-privilege OIDC roles and SHA-pinning in general: don't trust a name, trust a specific, verifiable artifact.

## Local development

```bash
pip install -r requirements.txt
python app.py                 # runs the Flask app locally on port 5000
pytest                        # run the same tests CI runs
flake8 app.py --max-line-length=100   # run the same lint check CI runs
```

## Infrastructure this pipeline depends on

- An ECR repository (`githubactions-app`) in `us-east-1`.
- An IAM OIDC identity provider for `token.actions.githubusercontent.com`, and an IAM role (`github-actions-ecr-push`) trusting it, scoped to this repository.
- A single EC2 instance in `us-west-2` running Docker, registered with AWS Systems Manager (via an instance profile with `AmazonSSMManagedInstanceCore`), with its security group allowing inbound TCP on 5000 and 5001.
- GitHub repository secrets: `EC2_INSTANCE_ID`.
- GitHub Environments `staging` and `production`, each with `APP_PORT` and `CONTAINER_NAME` variables, and `production` configured with a required-reviewer protection rule.

## What this project demonstrates

This pipeline was built incrementally, one real concept at a time: writing workflow YAML from scratch, federating GitHub Actions to AWS without static credentials, building and pushing containers, deploying to EC2 through Systems Manager rather than open SSH, enforcing quality and security gates with real branch protection (verified by deliberately breaking it), and modeling a realistic staging/production split with human approval and automatic recovery. Several of the hardest lessons came from things going wrong in genuinely instructive ways — a multi-day cross-region AWS networking investigation, a GitHub Actions OIDC claim format change, and a live supply chain attack on a widely-used security scanning action — each of which is reflected in a specific, deliberate design decision above rather than smoothed over.
