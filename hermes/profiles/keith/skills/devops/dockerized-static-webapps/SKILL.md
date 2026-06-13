---
name: dockerized-static-webapps
description: "Containerize and deploy static HTML/CSS/JS web apps on a VPS using Docker, Nginx, and optional Compose/domain setup."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [docker, static-site, nginx, vps, deployment, compose]
---

# Dockerized Static Web Apps

Use this skill when the user wants to host a static web app from a Git repo or local upload on a VPS inside Docker instead of directly on the host.

## Reference docs

- `references/visual-hero-asset-updates.md` — checklist for replacing generated hero/background images: local static asset, dimension verification, cache busting, container rebuild, and desktop/mobile visual checks.

## User preference

For Keith, keep operational answers terse and action-oriented unless he asks for explanation. Do not ask him to paste credentials in chat; use deploy keys, SSH config, or manual GitHub UI steps.

## Workflow

1. Identify source:
   - public Git repo
   - private Git repo with deploy key
   - uploaded zip/files
   - generated from scratch
2. If the app is still on the user's local machine, do not request GitHub credentials in chat. Have the user publish the repo, add a deploy key if private, and provide only the repo URL/host alias.
3. Identify app type:
   - plain static HTML/CSS/JS: serve directly with Nginx
   - Vite/React/etc.: build in a Node stage, then serve `dist/`
   - Next.js/static export: verify output mode before choosing image
3. Identify app type:
   - plain static HTML/CSS/JS: serve directly with Nginx
   - Vite/React/etc.: build in a Node stage, then serve `dist/`
   - Next.js/static export: verify output mode before choosing image
4. Check VPS prerequisites:
   - Docker Engine
   - Docker Compose plugin
   - open/listening ports and reverse proxy situation
5. If Docker is missing and the user authorizes installing it, install Docker first and verify with `docker run --rm hello-world` before discussing app container details.
6. Add container files:
   - `Dockerfile`
   - `nginx.conf`
   - `docker-compose.yml`
   - optionally `.dockerignore`
5. Build and run with Compose.
6. Verify with `docker ps`, logs, and an HTTP request to the exposed port/domain.
7. If using a domain, add a reverse proxy/HTTPS layer after the container works locally.

## Plain static template

`Dockerfile`:

```dockerfile
FROM nginx:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY . /usr/share/nginx/html
EXPOSE 80
```

`nginx.conf`:

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|svg|ico|woff|woff2)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

`docker-compose.yml`:

```yaml
services:
  app:
    build: .
    container_name: static-webapp
    restart: unless-stopped
    ports:
      - "8080:80"
```

## Docker install on Ubuntu VPS

When Docker is not installed and the user authorizes installation, use Docker's official apt repository, not random install scripts:

```bash
apt-get update
apt-get install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
. /etc/os-release
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu ${VERSION_CODENAME} stable" > /etc/apt/sources.list.d/docker.list
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker
```

Verify:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

## Private GitHub repo access

Do not request tokens in chat. Tell the user to create a repo deploy key manually:

```bash
ssh-keygen -t ed25519 \
  -f /root/.ssh/<project>_deploy_ed25519 \
  -C "<project>@$(hostname)" \
  -N ""
cat /root/.ssh/<project>_deploy_ed25519.pub
```

Then add the public key in GitHub repo settings as a deploy key. For clone-only deployments, do not enable write access.

Add an SSH host alias:

```sshconfig
Host github.com-<project>
  HostName github.com
  User git
  IdentityFile /root/.ssh/<project>_deploy_ed25519
  IdentitiesOnly yes
```

Clone URL shape:

```text
git@github.com-<project>:OWNER/REPO.git
```

## Pitfalls

- Do not expose on port 80 if another service already owns it; use `8080:80` first, then add reverse proxy.
- For static password generators, avoid server-side secrets entirely; everything in the image is public to clients.
- Verify the app in Docker before configuring DNS/HTTPS.
- A successful `docker run hello-world` proves Docker daemon, networking, and image pulls work.
- For generated hero/background images, do not trust prompt words like “4K” or “8K”; inspect actual dimensions and verify the served asset. If you upscale, say it is an upscale, not a native high-res render. Use `references/visual-hero-asset-updates.md` for the full checklist.
