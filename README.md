# Student Registration System — CI/CD Pipeline

A Flask + MongoDB student registration app, deployed via a fully automated CI/CD pipeline using **GitHub Actions**, **Docker**, **Amazon ECR**, and **Amazon EC2**, with email notifications on every run.

Forked from [mohanDevOps-arch/flask_Practice](https://github.com/mohanDevOps-arch/flask_Practice) as part of the PPMCAD CI/CD Pipeline assignment.

---

## Architecture

```
Developer push to main
        │
        ▼
┌───────────────┐    ┌──────────┐    ┌─────────────┐    ┌──────────────┐
│ GitHub Actions│ →  │  Test    │ →  │ Build image │ →  │ Push to ECR  │
│    trigger    │    │ (pytest) │    │  (Docker)   │    │              │
└───────────────┘    └──────────┘    └─────────────┘    └──────┬───────┘
                                                                 │
                                                                 ▼
                                                  ┌───────────────────────┐
                                                  │     Deploy to EC2:    │
                                                  │  - SSH into instance  │
                                                  │  - docker pull (ECR)  │
                                                  │  - stop/remove old    │
                                                  │  - run new container  │
                                                  │  - poll /health       │
                                                  └───────────┬───────────┘
                                                                 │
                                                                 ▼
                                                  ┌───────────────────────┐
                                                  │  Email notification:  │
                                                  │  success or failure   │
                                                  └───────────────────────┘
```

## Tech Stack

- **Backend**: Python, Flask
- **Database**: MongoDB Atlas (via Flask-PyMongo)
- **Containerization**: Docker
- **Registry**: Amazon ECR
- **Compute**: Amazon EC2 (Amazon Linux 2023)
- **CI/CD**: GitHub Actions
- **Notifications**: SMTP (Gmail)

---

## Prerequisites (set up manually before the pipeline runs)

### 1. MongoDB Atlas
- Free-tier (M0) cluster created
- Database user with read/write access
- Network Access set to allow the EC2 instance's IP (or `0.0.0.0/0` for simplicity on a free-tier learning project)
- Connection URI includes an explicit database name in the path (e.g. `.../student_db?...`) — without this, `flask_pymongo` cannot resolve `mongo.db` and every route fails with `AttributeError: 'NoneType' object has no attribute 'students'`

### 2. Amazon ECR
- Repository `flask-practice` created in the target region

### 3. Amazon EC2
- Amazon Linux 2023 instance
- Docker installed and running (`dnf install -y docker`, `systemctl enable --now docker`)
- AWS CLI v2 installed (needed for ECR authentication during deploy)
- IAM role attached: `AmazonEC2ContainerRegistryReadOnly`
- `ec2-user` added to the `docker` group so the deploy step doesn't need `sudo`
- Security group rules:
  - **Port 22 (SSH)** — restricted to a specific IP after initial setup (see Cleanup section)
  - **Port 5000 (App)** — open to `0.0.0.0/0`, required because GitHub Actions runners use dynamic, non-fixed IP addresses, so the health check step cannot be scoped to a single IP

### 4. GitHub Secrets
Stored under **Settings → Secrets and variables → Actions → Repository secrets** (not Environment secrets — see Failures Encountered below for why this distinction mattered):

| Secret | Purpose |
|---|---|
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | Pipeline's AWS access to push images to ECR |
| `ECR` login and image push | via above |
| `EC2_HOST` | Public IP/DNS of the EC2 instance |
| `EC2_USER` | SSH username (`ec2-user`) |
| `EC2_SSH_KEY` | Private key contents for SSH deploy |
| `MONGO_URI` | Full MongoDB Atlas connection string, including database name |
| `SMTP_SERVER` / `SMTP_PORT` / `SMTP_USER` / `SMTP_PASSWORD` | Email notification credentials |
| `NOTIFY_EMAIL` | Recipient address for pipeline alerts |

---

## Pipeline Stages

Defined in [`.github/workflows/ci-cd.yml`](.github/workflows/ci-cd.yml), triggered on every push to `main`:

1. **Checkout** — pulls latest source
2. **Install** — `pip install -r requirements.txt`
3. **Test** — runs `pytest`; pipeline stops here if any test fails (build/deploy jobs never start)
4. **Build** — Docker image tagged with the short Git commit SHA (not `latest`)
5. **Push to ECR** — authenticated push of the tagged image
6. **Deploy to EC2** — SSH in, pull the new image, stop/remove the old container, run the new one with `--restart unless-stopped`
7. **Verify** — polls `/health` up to 5 times with a delay; a container that starts but crashes is still reported as a failed deployment
8. **Notify** — separate success/failure email jobs with meaningfully different content (see below)

### Deploy method: SSH
Chosen over SSM because the assignment's reference architecture assumes a single always-on EC2 instance and SSH is simpler to wire into GitHub Actions without additional IAM/SSM agent configuration. The private key is stored only as a GitHub Secret, never committed.

---

## How to Reproduce a Deployment Manually

If the pipeline were unavailable:

```bash
# Authenticate to ECR
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com

# Pull the desired image tag
docker pull <account-id>.dkr.ecr.<region>.amazonaws.com/flask-practice:<commit-sha>

# Stop and remove the running container
docker stop flask-practice-app || true
docker rm flask-practice-app || true

# Run the new container
docker run -d --name flask-practice-app \
  --restart unless-stopped \
  -p 5000:5000 \
  -e MONGO_URI="<mongo-uri>" \
  <account-id>.dkr.ecr.<region>.amazonaws.com/flask-practice:<commit-sha>

# Verify
curl http://localhost:5000/health
```

---

## Failures Encountered & Fixes

Documented here as evidence of the actual debugging process during this assignment, not just the final working state.

| # | Symptom | Root Cause | Fix |
|---|---|---|---|
| 1 | `ModuleNotFoundError: No module named 'pymongo'` on local test | pymongo not installed in the shell environment used for testing | `pip3 install pymongo` |
| 2 | `ConfigurationError: The "dnspython" module must be installed to use mongodb+srv://` | `mongodb+srv://` URIs require `dnspython` for SRV DNS lookup | `pip3 install dnspython`, added to `requirements.txt` |
| 3 | `OperationFailure: bad auth : authentication failed` | Stale/mismatched Atlas database user credentials | Created a fresh Atlas database user with a known password rather than continuing to debug the original one |
| 4 | `pymongo.errors.InvalidURI: Invalid URI scheme` in GitHub Actions Test job | `MONGO_URI` secret was stored as an **Environment secret**, not a **Repository secret** — environment secrets are only injected into jobs that explicitly declare `environment:`, which this workflow didn't, so `${{ secrets.MONGO_URI }}` resolved to an empty string | Re-added all secrets under **Repository secrets** instead |
| 5 | `AttributeError: 'NoneType' object has no attribute 'students'` in pytest | `MONGO_URI` had no database name in the path (e.g. ended in `.net/?appName=...` with nothing between `/` and `?`), so `flask_pymongo` couldn't resolve `mongo.db` | Added an explicit database name to the URI (`.../student_db?...`) and created the database via Atlas Data Explorer |
| 6 | Deploy job succeeded but health check step failed and retried until timeout | EC2 security group had **no inbound rule for port 5000** at all — only port 22 was open, so the container was healthy locally but unreachable from GitHub's runners | Added inbound rule: Custom TCP, port 5000, source `0.0.0.0/0` |
| 7 | Docker not installed after EC2 instance issue | `systemctl enable docker` failed because the `docker` package was never actually installed (install command either wasn't run or errored silently) | Re-ran `dnf install -y docker` explicitly, confirmed with `rpm -qa \| grep docker` before enabling the service |
| 8 | Outlook SMTP `535 5.7.139 Authentication unsuccessful, basic authentication is disabled` | Microsoft disabled SMTP basic auth account-wide; an app password does not bypass this, since AUTH itself is blocked before credentials are checked | Switched to Gmail SMTP with an app password, which still supports SMTP AUTH |

---

## Screenshots (`/screenshots`)

| File | Shows |
|---|---|
| `01-pipeline-success.png` | Full pipeline run, all jobs green |
| `02-build-image-tag.png` | Docker image tagged with commit SHA |
| `03-ecr-repository.png` | Image pushed to ECR with matching tag |
| `04-deploy-container-running.png` | `docker ps` on EC2 showing the live container |
| `05-health-check-success.png` | `/health` endpoint returning a healthy, connected status |
| `06-success-email.png` | Success notification email with build details |
| `07-app-live-browser.png` | Student Registration UI running live |
| `08-failed-pipeline-test-stage.png` | Intentionally broken test halting the pipeline before build/deploy |
| `09-failure-email.png` | Failure notification email identifying the Test stage |

---

## Security Cleanup Performed

After the pipeline was confirmed working end to end:

- [ ] Restricted EC2 security group port 22 (SSH) to a specific IP instead of `0.0.0.0/0`; port 5000 intentionally left open to `0.0.0.0/0` since GitHub Actions runners don't use fixed IPs — documented here as a deliberate, known trade-off rather than an oversight
- [ ] Rotated the MongoDB Atlas database user password after testing, since an earlier password was exposed in plaintext during troubleshooting
- [ ] Rotated AWS IAM access keys used by the pipeline
- [ ] Confirmed `.env` was never committed to git history (`git log --all --full-history -- .env` returns nothing)
- [ ] Confirmed `.gitignore` excludes `.env`, `*.pem`, and `__pycache__/`
- [ ] Removed the unused `azure-pipelines.yml` left over from the original fork, since the assignment requires exactly one CI/CD tool (Jenkins or GitHub Actions, not both)

---

## Local Development Setup

```bash
git clone https://github.com/HellraiserOne/flask_Practice.git
cd flask_Practice
python -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env       # then fill in your own MONGO_URI
python app.py
```

Visit `http://localhost:5000`.

## Running Tests Locally

```bash
pytest -v
```

Tests run against the same `MONGO_URI` configured in `.env` — there is no separate isolated test database for this assignment; the `students` collection is cleared before and after each test run.