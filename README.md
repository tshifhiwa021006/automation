# Automation Demo

This repository provides a plug-and-play automation template to build and automatically deploy websites. It currently includes a GitHub Pages workflow (Auto Create & Deploy Website). I added an optional SSH-based deployment workflow and server-side helper files so your boss can set up deployments from scratch by following the steps below.

---

## What this repo does

- Builds and deploys a simple auto-generated site to GitHub Pages on push to `main` (existing workflow: `.github/workflows/deploy.yml`).
- Optional: deploy to your own server via SSH using the new workflow `.github/workflows/ssh-deploy.yml` and the provided server-side helper scripts.

---

## Quick decision checklist (pick one)

- If you want to host on GitHub Pages: no server required — the existing workflow already deploys the site.
- If you want to deploy to your own server (recommended for custom websites, Node/PHP/Python apps, or Docker): follow the "Server (SSH) deployment" instructions below.

---

## Files added for SSH server auto-deploy (plug-and-play)

- .github/workflows/ssh-deploy.yml  - GitHub Actions workflow that SSHes to your server and runs the deploy script.
- scripts/deploy.sh                 - Deploy helper script (run on the server) that pulls latest, detects app type (Docker / Node / static), installs deps, builds, and restarts services.
- deploy/myapp.service              - Example systemd unit for a Node app. Customize `ExecStart` and `WorkingDirectory` when you set it up on the server.
- deploy/docker-compose.yml.example - Example docker-compose file to run a website with nginx or a Node backend.

---

## Prerequisites (server)

These instructions assume an Ubuntu LTS server (22.04+). Adjust package manager commands if you use a different distro.

- SSH access to the server (you will add a deploy public key)
- sudo privileges for setup steps (the `deploy` user will not need sudo for normal deploys)
- git installed on server
- Optionally: Docker & docker-compose if you prefer containerized deployments

---

## Step-by-step: Prepare the server (one-time)

Run these commands locally (replace values where indicated).

1) Create an SSH key pair (on your laptop / CI runner) that GitHub Actions will use to connect:

```bash
# generate a keypair (do NOT add a passphrase if you want non-interactive CI deploys)
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ./automation_deploy_key -N ""
# this creates automation_deploy_key (private) and automation_deploy_key.pub (public)
```

2) On the server (first SSH manually as an admin): create a deploy user and add the public key

```bash
# on server (as root or sudo user)
sudo adduser --disabled-password --gecos "" deploy
sudo mkdir -p /home/deploy/.ssh
sudo chown -R deploy:deploy /home/deploy/.ssh
# append your public key from automation_deploy_key.pub
sudo sh -c 'cat >> /home/deploy/.ssh/authorized_keys' < automation_deploy_key.pub
sudo chmod 700 /home/deploy/.ssh
sudo chmod 600 /home/deploy/.ssh/authorized_keys
```

3) Create the deployment target directory and set ownership

```bash
sudo mkdir -p /var/www/myapp
sudo chown -R deploy:deploy /var/www/myapp
```

4) Install required packages (git + optional runtime)

For a simple static site or Node app:

```bash
sudo apt update
sudo apt install -y git curl
# Node (optional)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
```

For Docker deployments (optional):

```bash
# Docker
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
# allow deploy user to run docker (optional)
sudo usermod -aG docker deploy
```

5) (Optional) Clone the repo as the deploy user to verify permissions

```bash
sudo -u deploy git clone git@github.com:tshifhiwa021006/automation.git /var/www/myapp
```

6) Place the provided systemd unit (if using Node/systemd)

```bash
# as root or sudo
sudo cp deploy/myapp.service /etc/systemd/system/myapp.service
sudo systemctl daemon-reload
sudo systemctl enable --now myapp.service
```

Note: edit `/etc/systemd/system/myapp.service` to set `WorkingDirectory` and `ExecStart` to match your app.

---

## Step-by-step: Configure GitHub repository (one-time)

1) In the repo settings -> Secrets -> Actions, add the following secrets:

- SSH_PRIVATE_KEY  — the *private* key you generated (automation_deploy_key)
- DEPLOY_HOST      — server IP or hostname (e.g., `203.0.113.10`)
- DEPLOY_USER      — `deploy` (or another user name you chose)
- DEPLOY_PATH      — the path on the server where the repo is cloned (e.g., `/var/www/myapp`)
- DEPLOY_PORT      — (optional) SSH port, default 22 if omitted

How to add a secret: Repository -> Settings -> Secrets and variables -> Actions -> New repository secret.

2) (Optional) Ensure the deploy user's `authorized_keys` contains the public key (automation_deploy_key.pub).

---

## How the SSH deploy workflow works

- On push to `main`, `.github/workflows/ssh-deploy.yml` will run.
- The workflow checks out the repo, starts an SSH agent and loads the private key from the `SSH_PRIVATE_KEY` secret, then SSHes to `${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }}` and runs `scripts/deploy.sh` from the configured `DEPLOY_PATH`.
- The `scripts/deploy.sh` script (provided) pulls the latest, detects Docker/Node/static sites, runs the appropriate build commands, and restarts services (docker compose or systemd).

---

## Testing the setup

1) Verify you can SSH from your machine using the private key:

```bash
ssh -i ./automation_deploy_key deploy@203.0.113.10
```

2) From the server as the deploy user, verify you can pull the repo:

```bash
sudo -u deploy -i git -C /var/www/myapp pull origin main
```

3) Push a trivial change to `main` (e.g., update README) and watch GitHub Actions run both the Pages workflow and the SSH deploy workflow (check Actions tab). The SSH workflow output shows the remote commands and `deploy.sh` logs.

---

## Files you may want to customize

- scripts/deploy.sh — edit the build & restart steps to match your project's needs (npm/yarn/pip, build step, env variables).
- deploy/myapp.service — set the correct `ExecStart` for your app (Node, Python, etc.).
- deploy/docker-compose.yml.example — adapt ports, volumes, and image names.

---

## Security notes

- Keep the private key secret. Only store it in the GitHub Actions secrets (DO NOT commit private keys to the repo).
- Use a dedicated `deploy` user with limited privileges.
- If you allow the deploy user to restart system services, lock down which commands are allowed via sudoers if needed.
- Optionally restrict the public key in `authorized_keys` to only allow commands from GitHub Actions by using a forced command wrapper.

---

If you want I can:
- Commit the SSH deploy workflow and helper files into this repository now (I already added them in this change).
- Or instead produce a one-page PDF or plain text you can hand to your boss.

What I did now: I verified the existing GitHub Pages workflow is present and added an SSH-deploy workflow plus server helper scripts and updated this README with exact setup steps so your boss can set up the server and GitHub secrets and get plug-and-play deployments.
