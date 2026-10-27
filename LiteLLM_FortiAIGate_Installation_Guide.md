# LiteLLM on Ubuntu: Docker Compose + FortiAIGate 8.0.2

A GitHub-ready guide for deploying **LiteLLM 1.106.4** with **PostgreSQL 16**, Admin UI, and FortiAIGate as an OpenAI-compatible upstream provider.

> **Status:** LiteLLM API and Admin UI were accessed in the lab. Verify the complete FortiAIGate inference route in your environment.
>
> **Security:** Do not commit `.env` or real API keys. Rotate any keys that were previously exposed.

## Architecture

```text
Open WebUI / Client
       |
       v
LiteLLM Proxy :4000 -------- PostgreSQL 16 (private Docker network)
       |
       v
FortiAIGate 8.0.2 :30443
       |
       v
OpenAI-compatible model
```

## 1. Prerequisites

- Ubuntu with Docker Engine and Docker Compose plugin (`docker --version`, `docker compose version`).
- FortiAIGate API key and model `openai-demo-gpt-4.1-mini`.
- LiteLLM container can reach `faig2.cbcgslb.fortiseahk.com:30443`.
- Trusted TLS certificate for the FortiAIGate hostname.
- Optional OpenAI API key for a direct `gpt-4o` model.

## 2. Create project

```bash
mkdir -p ~/litellm
cd ~/litellm
```

Project files: `.env`, `.env.example`, `.gitignore`, `docker-compose.yml`, and `litellm_config.yaml`.

## 3. Generate master key and PostgreSQL password

Generate each value:

```bash
echo "LITELLM_MASTER_KEY=sk-$(openssl rand -hex 32)"
echo "POSTGRES_PASSWORD=$(openssl rand -hex 32)"
```

For a **new deployment only**, create `.env`:

```bash
cd ~/litellm
cat > .env <<'ENV_TEMPLATE'
LITELLM_MASTER_KEY=replace-with-generated-sk-key
POSTGRES_PASSWORD=replace-with-generated-password
OPENAI_API_KEY=your-openai-api-key
FORTIAIGATE_API_KEY=your-fortiaigate-api-key
ENV_TEMPLATE
chmod 600 .env
```

Replace placeholders with your generated values. **Do not overwrite an existing `.env`** when updating a working deployment. Changing the Compose PostgreSQL password after initialization does not change the stored database user's password.

Create `.env.example` with the same placeholder values for GitHub, **never real keys**.

## 4. Create `litellm_config.yaml`

```yaml
model_list:
  # Optional direct OpenAI connection
  - model_name: gpt-4o
    litellm_params:
      model: openai/gpt-4o
      api_key: os.environ/OPENAI_API_KEY

  # FortiAIGate as upstream OpenAI-compatible provider
  - model_name: fortiaigate-gpt-4.1-mini
    litellm_params:
      model: openai/openai-demo-gpt-4.1-mini
      api_base: https://faig2.cbcgslb.fortiseahk.com:30443/ai/openai/v1
      api_key: os.environ/FORTIAIGATE_API_KEY

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

The `openai/` prefix selects the OpenAI-compatible adapter. The client sends model `fortiaigate-gpt-4.1-mini`; FortiAIGate receives model `openai-demo-gpt-4.1-mini`. **Do not append `/chat/completions` to `api_base`.** Remove the direct OpenAI model if you do not need it.

## 5. Create `docker-compose.yml`

```yaml
services:
  postgres:
    image: postgres:16
    container_name: litellm-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: litellm
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD}
      POSTGRES_DB: litellm
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U litellm -d litellm"]
      interval: 10s
      timeout: 5s
      retries: 10

  litellm:
    image: ghcr.io/berriai/litellm:1.106.4
    container_name: litellm
    restart: unless-stopped
    ports:
      - "4000:4000"
    env_file:
      - .env
    environment:
      LITELLM_MASTER_KEY: ${LITELLM_MASTER_KEY:?Set LITELLM_MASTER_KEY}
      DATABASE_URL: postgresql://litellm:${POSTGRES_PASSWORD}@postgres:5432/litellm
      STORE_MODEL_IN_DB: "true"
    volumes:
      - ./litellm_config.yaml:/app/config.yaml:ro
    depends_on:
      postgres:
        condition: service_healthy
    command:
      - "--config"
      - "/app/config.yaml"
      - "--port"
      - "4000"
      - "--host"
      - "0.0.0.0"

volumes:
  postgres_data:
```

PostgreSQL is only accessible on the Docker network. Generated hexadecimal passwords avoid URL-encoding problems in `DATABASE_URL`. For production, restrict access to port 4000 or front it with an HTTPS reverse proxy. If LiteLLM logs report database migration/schema errors, use the migration procedure appropriate to this exact image version.

## 6. Start and inspect

```bash
cd ~/litellm
docker compose config --quiet
docker compose up -d
docker compose ps
docker compose logs --tail=100 litellm
```

Successful startup includes `Application startup complete` and `Uvicorn running on http://0.0.0.0:4000`.

## 7. Admin UI

Browse to `http://<UBUNTU_IP>:4000/ui`.

| Field | Initial value |
|---|---|
| Username | `admin` |
| Password | `LITELLM_MASTER_KEY` value from `.env` |

If the UI reports **`Authentication Error, Not connected to DB!`**, inspect PostgreSQL, `DATABASE_URL`, and database initialization. After creating and verifying an individual administrator account, consider setting `general_settings.disable_env_credential_login: true` to disable shared environment-based login.

## 8. Verify API

```bash
cd ~/litellm
set -a
source .env
set +a

curl -sS http://localhost:4000/v1/models \
  -H "Authorization: Bearer ${LITELLM_MASTER_KEY}" \
  | python3 -m json.tool
```

Check that `fortiaigate-gpt-4.1-mini` appears. Then test inference:

```bash
curl -i http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${LITELLM_MASTER_KEY}" \
  -d '{
    "model": "fortiaigate-gpt-4.1-mini",
    "messages": [{"role": "user", "content": "Hello, what is your name?"}],
    "max_tokens": 100
  }'
```

LiteLLM should call `https://faig2.cbcgslb.fortiseahk.com:30443/ai/openai/v1/chat/completions` using the upstream FortiAIGate API key. Confirm the request appears in FortiAIGate inspection logs.

## 9. Alternative: add FortiAIGate model in GUI

In **Models + Endpoints → Add Model** (LiteLLM 1.106.4):

| Field | Value |
|---|---|
| Provider | Custom / OpenAI-compatible |
| LiteLLM Model Name(s) | `openai/openai-demo-gpt-4.1-mini` |
| Public Model Name | `fortiaigate-gpt-4.1-mini` |
| Mode | Chat - `/chat/completions` |
| API Base | `https://faig2.cbcgslb.fortiseahk.com:30443/ai/openai/v1` |
| API Key | FortiAIGate API key |

Exact field names may vary. Avoid defining the same alias in both GUI and YAML unless you intend multiple deployments.

## 10. TLS and troubleshooting

Your original FortiAIGate curl used `-k` (skip TLS verification). LiteLLM does not inherit this behavior. Prefer a trusted certificate or installing the issuing CA in the container.

```bash
curl -Iv https://faig2.cbcgslb.fortiseahk.com:30443/ai/openai/v1/chat/completions
```

| Symptom | What to check |
|---|---|
| Container exits | `docker compose logs litellm` |
| `UnsafeMasterKeyError` | `LITELLM_MASTER_KEY` exists and is valid |
| 401 | LiteLLM bearer token or FortiAIGate upstream token |
| `No connected db` | PostgreSQL connectivity and auth/database initialization |
| Model not found | `litellm_config.yaml`, `/v1/models` |
| TLS error | CA trust, hostname, certificate chain |
| Timeout | DNS, firewall, TCP 30443, container egress |

```bash
docker compose ps
docker compose logs -f litellm
docker compose logs --tail=100 postgres
docker compose up -d --force-recreate litellm
docker compose down
```

**Avoid `docker compose down -v`** unless intentionally deleting the PostgreSQL database volume.

## 11. Publish safely to GitHub

Create `.gitignore`:

```gitignore
.env
.env.*
!.env.example
*.log
__pycache__/
.DS_Store
```

Save this guide as `README.md` (or keep its current name). Confirm files contain no secrets before committing:

```bash
cd ~/litellm
git init
git add README.md docker-compose.yml litellm_config.yaml .env.example .gitignore
git diff --cached --check
git status
git commit -m "Document LiteLLM FortiAIGate deployment"
git branch -M main
git remote add origin https://github.com/<OWNER>/<REPOSITORY>.git
git push -u origin main
```

Replace `<OWNER>/<REPOSITORY>` with your actual GitHub repository path. If you keep the guide under its original filename, change the `git add README.md` argument accordingly.

## Validation checklist

- [ ] `.env` configured and excluded from Git
- [ ] PostgreSQL healthy
- [ ] LiteLLM running on port 4000
- [ ] Admin UI login succeeds
- [ ] `/v1/models` shows FortiAIGate alias
- [ ] Chat completion succeeds through FortiAIGate
- [ ] FortiAIGate security logs show request
- [ ] No credentials committed to GitHub
