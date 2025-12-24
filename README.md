# GitLab on Docker Compose (QNAP-friendly)

This repo contains a production-oriented Docker Compose setup for GitLab EE with:
- GitLab Omnibus container (bind-mounted persistence)
- GitLab Runner (Docker executor) with instance runner auto-registration
- HTTP-only GitLab in-container (no TLS)
- Optional reverse proxy components: Cloudflare Tunnel (cloudflared) and Traefik (disabled by default)
- Optional Microsoft Entra ID (Azure AD v2) OmniAuth (`azure_activedirectory_v2`).

## Files

- `docker-compose.yml`: main stack
- `.env.example`: environment variables template

## Quick start (normal Docker Compose)

1. Create `.env`:

   ```bash
   cp .env.example .env
   ```

2. Edit `.env`:

   - `GITLAB_EXTERNAL_URL`
   - `GITLAB_HTTP_BIND`, `GITLAB_HTTP_PORT`
   - `GITLAB_SSH_BIND`, `GITLAB_SSH_PORT`
   - `GITLAB_RUNNER_REGISTRATION_TOKEN`

3. Start (core services):

   ```bash
   docker compose up -d
   ```

## Optional profiles

This compose file includes optional services that are disabled by default and must be enabled with a profile.

### Traefik (disabled by default)

```bash
docker compose --profile traefik up -d
```

### Cloudflared (disabled by default)

```bash
docker compose --profile cloudflared up -d
```

Cloudflared is configured to run with a tunnel token (`CLOUDFLARED_TUNNEL_TOKEN`). Configure public hostnames/routes in the Cloudflare Zero Trust dashboard.

## QNAP (Container Station) notes

Container Station typically supports Compose stacks, but the UI varies by firmware.

Recommended approach:

1. Upload this folder to QNAP.
2. Create `.env` next to `docker-compose.yml`.
3. Deploy the compose stack.

If Container Station does not honor `--env-file`, place a `.env` file in the same directory as `docker-compose.yml` (this stack uses `env_file: ./.env` per service).

## Microsoft Entra ID (Azure AD v2) login

GitLab supports the OmniAuth provider name `azure_activedirectory_v2`.

### 1) Azure app registration

In Azure Portal:

- Register an application.
- Add a Redirect URI (Web):

  `https://<your-gitlab-host>/users/auth/azure_activedirectory_v2/callback`

- Note:
  - Tenant ID
  - Client ID
  - Client Secret

GitLab docs: https://docs.gitlab.com/ee/integration/azure.html

### 2) Configure `.env`

Set:

```dotenv
AZURE_AD_ENABLED=true
AZURE_AD_CLIENT_ID=...
AZURE_AD_CLIENT_SECRET=...
AZURE_AD_TENANT_ID=...

# Optional
AZURE_AD_LABEL=Azure AD v2
AZURE_AD_SCOPE=openid profile email
AZURE_AD_BASE_AZURE_URL=
```

### 3) Restart GitLab

```bash
docker compose up -d
```

## SMTP (outbound email)

Set these in `.env` and restart GitLab:

```dotenv
SMTP_ENABLED=true
SMTP_ADDRESS=smtp.example.com
SMTP_PORT=587
SMTP_USERNAME=...
SMTP_PASSWORD=...
SMTP_DOMAIN=example.com
SMTP_AUTHENTICATION=login
SMTP_ENABLE_STARTTLS_AUTO=true
SMTP_TLS=false
SMTP_OPENSSL_VERIFY_MODE=peer

GITLAB_EMAIL_FROM=gitlab@example.com
GITLAB_EMAIL_REPLY_TO=gitlab@example.com
```

Then:

```bash
docker compose up -d
```

## GitLab Runner (instance) auto-registration

This stack starts `gitlab-runner` and registers it automatically on first start if no runner exists in the runner config volume.

You must set the instance registration token:

- GitLab UI: Admin Area → CI/CD → Runners → Registration token
- `.env`: `GITLAB_RUNNER_REGISTRATION_TOKEN=...`

### Docker-in-Docker (DinD) for CI builds

Recommended CI pattern: use `docker:dind` as a job service.

Example `.gitlab-ci.yml`:

```yaml
image: docker:27

services:
  - name: docker:27-dind
    command: ["--tls=false"]

variables:
  DOCKER_HOST: tcp://docker:2375
  DOCKER_TLS_CERTDIR: ""

build:
  script:
    - docker version
    - docker build -t myapp:ci .
```
