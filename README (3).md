# Live Update from Main to Production

Automatically deploy a Flask app to an AWS EC2 instance every time code is pushed to the `main` branch, using Docker and a GitHub Actions self-hosted runner.

## How it works

![CI/CD pipeline: push to main, GitHub Actions, runner on EC2 builds image and replaces the container](docs/pipeline.svg)

1. A developer pushes a commit to `main`.
2. GitHub triggers the **Deploy** workflow.
3. The self-hosted runner on EC2 picks up the job and checks out the code.
4. `docker build` creates a new image.
5. The old container is removed and a new one starts from the new image.
6. The updated app is live at `http://<ec2-public-ip>:9000`.

## Tech stack

- Python / Flask
- Docker
- GitHub Actions (self-hosted runner)
- AWS EC2 (Amazon Linux 2023)

## Project structure

```
.
├── .github/
│   └── workflows/
│       └── deploy.yaml      # CI/CD pipeline
├── docs/
│   └── pipeline.svg         # Animated pipeline diagram
├── app.py                   # Flask application
├── Dockerfile               # Container image definition
├── requirements.txt         # Python dependencies
└── README.md
```

## Prerequisites

- An AWS EC2 instance (Amazon Linux 2023) with SSH access
- A GitHub account and repository
- EC2 security group allowing inbound TCP **9000** (app) and **22** (SSH)

---

## Step-by-step setup

### Step 1: Prepare the EC2 instance

Install Docker and Git, and let `ec2-user` run Docker without `sudo`:

```bash
sudo dnf install -y docker git
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
```

Log out and back in so the group change takes effect, then check:

```bash
docker ps
```

### Step 2: Create the application

**app.py**

```python
from flask import Flask

app = Flask(__name__)


@app.route("/")
def home():
    return "Sample test"


@app.route("/health")
def health():
    return {"status": "UP"}


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

The app must listen on `0.0.0.0`, otherwise it is unreachable from outside the container.

**requirements.txt**

```
flask
```

**Dockerfile** (example; adjust to your project)

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python3", "app.py"]
```

### Step 3: Push the code to GitHub

1. Create an **empty** repository on GitHub (no README or .gitignore).
2. Create a `.gitignore` and push from EC2:

```bash
cd ~
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

printf "app.log\n.env\n*.pem\n" > .gitignore

git init -b main
git add app.py Dockerfile requirements.txt .gitignore
git commit -m "initial commit"
git remote add origin https://github.com/<your-user>/<your-repo>.git
git push -u origin main
```

3. When Git asks for a password, use a **personal access token** (GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)) with the **repo** scope. Add the **workflow** scope too, because pushing files under `.github/workflows/` requires it.

Never paste a token into a chat, a file in the repo, or a screenshot.

### Step 4: Install the GitHub Actions self-hosted runner

The runner is a small agent on your EC2 instance that receives jobs from GitHub, so no SSH or inbound access from GitHub is needed.

1. In your repo go to **Settings → Actions → Runners → New self-hosted runner**.
2. Choose **Linux** and **x64** (check with `uname -m`; use ARM64 if it prints `aarch64`).
3. Run the **Download** commands shown on that page on your EC2 instance.
4. Install the ICU library, which Amazon Linux 2023 does not include and the runner needs:

```bash
sudo dnf install -y libicu
```

5. Copy the `./config.sh ...` line from the **Configure** section of the same page and run it from `~/actions-runner`:

```bash
./config.sh --url https://github.com/<your-user>/<your-repo> --token <REGISTRATION_TOKEN>
```

   - The registration token comes from that page. It is **not** your personal access token (`ghp_...`), and it expires after about an hour.
   - At the **runner group** prompt, press Enter (Default). Give the runner any name. Accept the defaults for labels and work folder.

6. Install it as a service so it survives logouts and reboots:

```bash
cd ~/actions-runner
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status
```

Status should show `active (running)`, and the runner should appear as **Idle** under **Settings → Actions → Runners**.

> Do not start the runner with `./run.sh` for normal use. It stops when the terminal closes and jobs stay **Queued**.

### Step 5: Add the deployment workflow

Create `.github/workflows/deploy.yaml`:

```yaml
name: Deploy

on:
  push:
    branches: [main]

concurrency:
  group: deploy
  cancel-in-progress: true

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4

      - name: Rebuild and restart container
        run: |
          docker build --pull -t appsampleimage .
          docker rm -f mysamplecontainer || true
          docker run -d --name mysamplecontainer -p 9000:5000 appsampleimage
```

What each part does:

| Part | Purpose |
|------|---------|
| `on.push.branches: [main]` | Triggers only on pushes to `main` |
| `concurrency` | Cancels older queued deploys when a newer push arrives |
| `runs-on: self-hosted` | Runs the job on your EC2 runner |
| `actions/checkout@v4` | Fetches the latest code |
| `docker build --pull` | Builds a new image and refreshes the base image |
| `docker rm -f ... \|\| true` | Removes the old container (does not fail if none exists) |
| `docker run -d ... -p 9000:5000` | Starts the new container; host port 9000 maps to container port 5000 |

YAML is indentation-sensitive. All three `docker` lines must start at the same column, using spaces only.

Commit and push:

```bash
cd ~
git add .github/workflows/deploy.yaml
git commit -m "add deploy workflow"
git push
```

### Step 6: Open the port

In the EC2 **security group**, add an inbound rule: **Custom TCP, port 9000**, from the sources that should reach the app.

### Step 7: Verify

1. Open the repo's **Actions** tab. The latest **Deploy** run should be green.
2. On EC2:

```bash
docker ps
curl http://localhost:9000/
curl http://localhost:9000/health
```

3. In a browser, open `http://<ec2-public-ip>:9000`.

Expected responses:

- `/` returns `Sample test`
- `/health` returns `{"status": "UP"}`

---

## Daily workflow

1. Edit the code (locally, in GitHub's web editor, or on EC2).
2. Commit and push to `main`.
3. GitHub Actions rebuilds and redeploys automatically, usually within 10 to 15 seconds.
4. Refresh the page to see the change.

If you edit in more than one place (for example the GitHub web editor and EC2), run this before pushing from EC2 to avoid a "non-fast-forward" rejection:

```bash
git pull --rebase origin main
```

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Run stays **Queued** | Runner is offline | `cd ~/actions-runner && sudo ./svc.sh start`, and check **Settings → Actions → Runners** |
| `config.sh` says libicu is missing | ICU library not installed | `sudo dnf install -y libicu` |
| `config.sh` returns 404 | Used a personal access token instead of the registration token | Copy a fresh token from the **New self-hosted runner** page |
| "Could not find any self-hosted runner group" | Typed a runner name at the group prompt | Press Enter for the Default group |
| Push rejected: needs `workflow` scope | Token lacks the scope | Add the **workflow** scope to the token |
| Push rejected: non-fast-forward | GitHub has commits EC2 lacks | `git pull --rebase origin main`, then `git push` |
| Run fails instantly, no steps | YAML indentation error | Make all lines inside `run: \|` use the same indentation |
| `port is already allocated` | Another container holds port 9000 | `docker ps`, then `docker rm -f <name>` |
| `permission denied` on Docker socket | Runner user not in the docker group | `sudo usermod -aG docker ec2-user`, then restart the runner service |
| Page does not load | Port closed or app bound to localhost | Open TCP 9000 in the security group; use `host="0.0.0.0"` |
| Old code still showing | Browser cache or old container | Hard refresh (`Ctrl+Shift+R`); check `docker ps` for a recent CREATED time |

Useful commands:

```bash
docker ps                          # running containers
docker logs mysamplecontainer      # app logs
sudo ./svc.sh status               # runner service status (from ~/actions-runner)
```

## Security notes

- Keep tokens out of the repo, chat, and screenshots. If one is exposed, delete it on GitHub and create a new one.
- A self-hosted runner executes workflow code on your server. Use a **private** repository, or under **Settings → Actions → General** require approval for workflows from outside contributors.
- Restrict port 22 and port 9000 in the security group to the sources that need them.

## Optional improvements

Restart the container automatically after a reboot or crash:

```bash
docker run -d --name mysamplecontainer --restart unless-stopped -p 9000:5000 appsampleimage
```

Remove old dangling images after each deploy (add as the last line of the `run` block):

```bash
docker image prune -f
```

Fail the deploy if the app does not start (add at the end of the `run` block):

```bash
sleep 3
curl -f http://localhost:9000/health
```
