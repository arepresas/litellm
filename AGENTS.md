# AGENTS.md

Operational runbook for AI agents installing and configuring this local LiteLLM gateway.

Source of truth: `README.md`. If this file and `README.md` disagree, `README.md` wins. Read its sections on model catalog, routing, auth, OAuth, OpenCode setup, and troubleshooting before changing anything.

## Context pointers

- Read `README.md` when changing models, routing, auth, OAuth, OpenCode config, or diagnosing failures.
- Read `env.example` when creating `.env`.
- Read `config.yaml` before editing deployments or `router_settings`.
- Read `docker-compose.yaml` before changing ports, mounts, or env wiring.

## Hard guardrails

- Keep secrets in shell variables and files with restrictive modes. Print only status codes, counts, and non-secret identifiers.
- Use `rtk` for `curl` and Docker commands so secrets stay out of terminal output.
- Use the master key only for admin routes (`/key/*`, `/credentials`, Admin UI). OpenCode receives only the virtual key.
- Use `/v1/responses` for all model calls. GPT/MAI Copilot models fail on `/v1/chat/completions`.
- Keep `LITELLM_SALT_KEY` stable after first boot. Rotating it breaks stored credentials.
- Leave `XAI_API_KEY` unset. `config.yaml` uses `use_xai_oauth: true`; an API key skips OAuth.
- Keep gateway and Admin UI bound to `127.0.0.1` unless a human approves TLS plus auth in front.
- Commit only when a human asks.

## Step 1 — Prepare `.env`

```bash
cp env.example .env
```

Generate values without printing them:

```bash
openssl rand -hex 24   # POSTGRES_PASSWORD
openssl rand -hex 32   # LITELLM_MASTER_KEY, LITELLM_SALT_KEY
```

Prefix both LiteLLM values with `sk-`. Replace every `CHANGE_ME` placeholder in `.env`.

Completion criterion: `.env` exists, has the five required variables, and `git check-ignore -v .env` reports an ignore rule.

## Step 2 — Start and health-check the stack

```bash
docker compose up -d
docker compose ps
rtk curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:4000/health/liveliness
```

Expected containers: `litellm`, `litellm-db`. Expected health code: `200`.

Validate the mounted config:

```bash
docker exec litellm litellm --config /app/config.yaml --validate_config
```

After any `config.yaml` edit:

```bash
docker compose restart litellm
```

Wait about 15 seconds before rechecking health. `000` immediately after restart means startup is still in progress.

Completion criterion: both containers are up, config validation reports no errors, and health returns `200`.

## Step 3 — Create or reuse the virtual key

Load env vars without printing them:

```bash
set -a; . ./.env; set +a
```

Reuse the existing OpenCode key when present:

```bash
export VKEY=$(python3 -c "import json; print(json.load(open('/home/overlag/.local/share/opencode/auth.json'))['litellm']['key'])")
```

If no key exists, generate one with the master key:

```bash
MODELS='["copilot-gpt-5.3-codex","copilot-gpt-6-luna","copilot-gpt-6.1-terra","copilot-gpt-6.2-astra","copilot-gpt-6.2-sol","copilot-mai-code-2","copilot-mai-code-2-flash","claude-copilot-haiku-5.5","claude-copilot-opus-5.5","claude-copilot-sonnet-5.5","copilot-kimi-k3","grok-4.20","grok-4.20-non-reasoning","grok-4.3","grok-4.7"]'

export VKEY=$(rtk curl -s -X POST http://127.0.0.1:4000/key/generate \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"alias":"Opencode","user_id":"default_user_id","models":'"$MODELS"',"rpm_limit":60,"tpm_limit":200000}' \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['key'])")
```

Inspect safely:

```bash
rtk curl -s -G --data-urlencode "key=$VKEY" \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  http://127.0.0.1:4000/key/info
```

Expected: alias `Opencode`, user `default_user_id`, 15 models, `rpm_limit=60`, `tpm_limit=200000`.

Completion criterion: `VKEY` is set in the shell and `/key/info` confirms the 15-model allowlist and limits.

## Step 4 — Log in to xAI OAuth

The callback server binds to `127.0.0.1` inside the process. A browser on the host cannot complete the loopback callback from the normal Compose container. Use a temporary host-network container and write the token to the mounted host directory.

Stop the gateway so host port `56121` is free:

```bash
docker compose stop litellm
```

From the repository root, run:

```bash
docker run --rm -it --network host --entrypoint python \
  -v "$PWD/xai_oauth:/root/.config/litellm/xai_oauth" \
  docker.litellm.ai/berriai/litellm:main-stable \
  -c 'from litellm.llms.xai.oauth import XAIOAuthAuthenticator; XAIOAuthAuthenticator().login(force=True)'
```

Callback URL: `http://127.0.0.1:56121/callback`. Keep the process open while authorizing in the browser. If no browser opens, use the URL printed by the command.

If xAI shows `Enter this code to finish signing in` / `Copy the code below into Grok Build`, LiteLLM does not read that code. With the same login process still running, open this URL locally using the code from that page and the `state` from the URL printed by that exact attempt:

```text
http://127.0.0.1:56121/callback?code=<CODE_FROM_PAGE>&state=<STATE_FROM_PRINTED_URL>
```

Never reuse codes/states from earlier attempts. Never print or paste them into chat.

After success, restart the gateway:

```bash
docker compose start litellm
```

Success text: `xAI OAuth login successful. Credentials saved to <auth_file>.`

Check file mode without reading contents:

```bash
stat -c '%a %n' xai_oauth/auth.json
```

Expected: `600`.

Completion criterion: login reports success, the gateway is running, and the host token file exists with mode `600`.

## Step 5 — Connect GitHub Copilot per-user OAuth

Preferred path: Admin UI at `http://127.0.0.1:4000/ui/` (port `56121` is only the xAI OAuth callback).

1. **Models + Endpoints** → **LLM Credentials** → **Add Credential**.
2. Provider `GitHub Copilot`, Credential Name `Github Copilot`, Auth Type `Per-user GitHub OAuth`.
3. Leave GitHub host empty for `github.com`.
4. Save, then **Your connections** → `Github Copilot` → **Connect**.
5. Open `https://github.com/login/device`, enter the code, approve with the Copilot-seat account.
6. Status becomes **Connected** with the GitHub login.

API alternative, credential creation with master key:

```bash
rtk curl -s -X POST http://127.0.0.1:4000/credentials \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"credential_name":"Github Copilot","credential_values":{"github_copilot_auth_type":"per_user_oauth"},"credential_info":{"custom_llm_provider":"github_copilot"}}'
```

Device flow with the virtual key:

```bash
rtk curl -s -X POST \
  http://127.0.0.1:4000/credentials/Github%20Copilot/user_connection/start \
  -H "Authorization: Bearer $VKEY"
```

Poll with the returned `flow_handle`:

```bash
rtk curl -s -X POST \
  http://127.0.0.1:4000/credentials/Github%20Copilot/user_connection/poll \
  -H "Authorization: Bearer $VKEY" \
  -H 'Content-Type: application/json' \
  -d '{"flow_handle":"<FLOW_HANDLE>"}'
```

Use the virtual key for the connection so it belongs to `default_user_id`. `lite credentials connect` is absent in this image.

Completion criterion: `/credentials` shows `Github Copilot` connected for the intended GitHub account.

## Step 6 — Configure OpenCode

Config path: `~/.config/opencode/opencode.json`. OpenCode 1.18.35 uses V1 shape: singular `provider`, `npm`, `options`.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "litellm/grok-4.20-non-reasoning",
  "provider": {
    "litellm": {
      "name": "LiteLLM",
      "npm": "@ai-sdk/openai",
      "options": {
        "baseURL": "http://127.0.0.1:4000/v1"
      },
      "models": {
        "grok-4.20": { "name": "grok-4.20" },
        "grok-4.20-non-reasoning": { "name": "grok-4.20-non-reasoning" },
        "copilot-gpt-6-luna": { "name": "copilot-gpt-6-luna", "temperature": false }
      }
    }
  }
}
```

Leave `options.apiKey` unset. Store the virtual key in OpenCode:

```text
/connect
→ Other
→ provider id: litellm
→ API key: <virtual key>
```

Verify:

```bash
opencode auth list
```

If an old credential is cached, restart the OpenCode process.

Completion criterion: `opencode auth list` shows the `litellm` credential and `/models` lists gateway models.

## Step 7 — End-to-end validation

Responses API with the virtual key:

```bash
rtk curl -s -X POST http://127.0.0.1:4000/v1/responses \
  -H "Authorization: Bearer $VKEY" \
  -H 'Content-Type: application/json' \
  -d '{"model":"grok-4.20-non-reasoning","input":"say hi"}'
```

Expected: HTTP `200`, `"status": "completed"`, text in `output[].content[].text`.

List visible models:

```bash
rtk curl -s -H "Authorization: Bearer $VKEY" http://127.0.0.1:4000/v1/models
```

Expected: 15 model IDs. `claude-copilot-fable-5.2` and `copilot-grok-4.7` are absent.

Blocked model check:

```bash
rtk curl -s -o /dev/null -w '%{http_code}\n' -X POST \
  http://127.0.0.1:4000/v1/responses \
  -H "Authorization: Bearer $VKEY" \
  -H 'Content-Type: application/json' \
  -d '{"model":"claude-copilot-fable-5.2","input":"hi"}'
```

Expected: `403`.

OpenCode smoke test:

```bash
opencode run -m litellm/grok-4.20-non-reasoning "hello" --print-logs --log-level DEBUG
```

Expected log line: `llm.runtime=ai-sdk llm.provider=litellm llm.model=grok-4.20-non-reasoning`.

Completion criterion: Responses call is `200`, model list has 15 IDs, hidden model returns `403`, and OpenCode smoke test succeeds.

## Step 8 — Spend check

```bash
set -a; . ./.env; set +a
rtk docker exec -e PGPASSWORD="$POSTGRES_PASSWORD" litellm-db \
  psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" \
  -c 'SELECT count(*) AS total, max("startTime") AS last_start FROM "LiteLLM_SpendLogs";'
```

Completion criterion: query returns a row without error. No custom callbacks are configured.

## Escalate to a human

- Browser or device-code approval is required for xAI or GitHub OAuth.
- A provider returns `401` after reconnect/re-login.
- A model is dead upstream or missing from the 15-model allowlist.
- Secret rotation is needed, especially `LITELLM_SALT_KEY`.
- Remote access, TLS, or non-local port binding is requested.
- `README.md` and runtime behavior disagree.

## Reference

- `README.md` — full architecture, model catalog, routing, troubleshooting, known limitations, and source links.
- `env.example` — env var template and generation hints.
- `config.yaml` — deployments, OAuth flags, hidden models, router settings.
- `docker-compose.yaml` — ports, mounts, healthcheck, env wiring.
- `.gitignore` — secret and local-state exclusions.
