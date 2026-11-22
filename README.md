# Local LiteLLM Gateway for OpenCode

Last verified: 2026-11-22. Stack: LiteLLM `main-stable`, Postgres 18, OpenCode 1.18.35.

This repository runs a local OpenAI-compatible gateway that gives OpenCode access to xAI Grok models through xAI OAuth and GitHub Copilot models through per-user GitHub OAuth. A virtual key, not the master key, is stored in OpenCode. Postgres holds virtual keys, Copilot user connections, and spend logs.

## Features

- One endpoint: `http://127.0.0.1:4000/v1/responses`.
- xAI OAuth for Grok models. No `XAI_API_KEY` needed.
- GitHub Copilot per-user OAuth. No shared Copilot token.
- Virtual key allowlist: 15 active models only.
- Per-key rate limits: `rpm_limit=60`, `tpm_limit=200000`.
- Retries, cooldown, and explicit fallbacks.
- Spend tracking in Postgres with no custom callbacks.
- OpenCode 1.18.35 legacy V1 provider config.

## Architecture

```text
OpenCode 1.18.35
  |  /v1/responses + virtual key
  v
LiteLLM gateway (127.0.0.1:4000)
  |-- config.yaml (read-only mount)
  |-- router_settings (retries, timeout, fallbacks)
  |-- xAI OAuth token (mounted ./xai_oauth/auth.json)
  |-- Copilot per-user credential (encrypted in Postgres)
  v
xAI API  |  GitHub Copilot API

Postgres (litellm-db)
  |-- virtual keys / users
  |-- Copilot user connections
  |-- LiteLLM_SpendLogs

Admin UI: http://127.0.0.1:4000/ui/ (port 56121 is only the xAI OAuth callback)
```

Request flow:

1. OpenCode sends `POST /v1/responses` to the gateway with the virtual key.
2. LiteLLM validates the key against its model allowlist and rate limits.
3. The router picks the deployment. `use_xai_oauth: true` selects the xAI OAuth path. `litellm_credential_name: Github Copilot` selects the Copilot per-user path.
4. On failure, retries and fallbacks apply.
5. Spend is written to `LiteLLM_SpendLogs`.

## Repository layout

| Path | Purpose |
| --- | --- |
| `docker-compose.yaml` | Gateway + Postgres services, ports, volumes, env wiring. |
| `config.yaml` | Model list, OAuth flags, credential names, router settings. Mounted read-only. |
| `.env` | Secrets. Never commit. Loaded by Compose. |
| `xai_oauth/auth.json` | xAI OAuth token. Never commit. Mounted into the container. |
| `postgres_data/` | Postgres data directory. Never commit. |

`.gitignore` already excludes `.env`, `postgres_data/`, and `xai_oauth/`.

## Prerequisites

- Docker with Compose.
- OpenCode 1.18.35.
- An xAI account with Grok access.
- A GitHub account with an active GitHub Copilot seat.
- Optional but recommended: `rtk` as a wrapper for `curl` and Docker commands. It redacts secrets from terminal output. Plain `curl` also works if you keep secrets out of logs.

Environment variables in `.env`:

| Variable | Used for |
| --- | --- |
| `POSTGRES_USER` | Postgres user. |
| `POSTGRES_PASSWORD` | Postgres password. |
| `POSTGRES_DB` | Postgres database. |
| `LITELLM_MASTER_KEY` | Admin key for `/key/*`, `/credentials`, and the Admin UI. |
| `LITELLM_SALT_KEY` | Encryption salt for stored credentials. |

Do not put `LITELLM_MASTER_KEY` in OpenCode. OpenCode uses a separate virtual key.

## Start and validate

```bash
docker compose up -d
docker compose ps
```

Expected containers: `litellm` and `litellm-db`.

Health check:

```bash
rtk curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:4000/health/liveliness
```

Expected output:

```text
200
```

Validate the mounted config without starting another server:

```bash
docker exec litellm litellm --config /app/config.yaml --validate_config
```

After editing `config.yaml`, restart the gateway:

```bash
docker compose restart litellm
```

Startup takes about 15 seconds. If health returns `000` immediately after restart, wait and check `docker logs litellm`.

## Model catalog

### Active models

| Group | Model IDs | Auth path |
| --- | --- | --- |
| xAI | `grok-4.20`, `grok-4.20-non-reasoning`, `grok-4.3`, `grok-4.7` | `use_xai_oauth: true` |
| Copilot Claude | `claude-copilot-haiku-5.5`, `claude-copilot-opus-5.5`, `claude-copilot-sonnet-5.5` | `litellm_credential_name: Github Copilot` |
| Copilot GPT/Codex | `copilot-gpt-5.3-codex`, `copilot-gpt-6-luna`, `copilot-gpt-6.1-terra`, `copilot-gpt-6.2-astra`, `copilot-gpt-6.2-sol` | `litellm_credential_name: Github Copilot` |
| Copilot other | `copilot-kimi-k3`, `copilot-mai-code-2`, `copilot-mai-code-2-flash` | `litellm_credential_name: Github Copilot` |

Total: 15 active models.

### Hidden models

| Model ID | Reason |
| --- | --- |
| `claude-copilot-fable-5.2` | Dead upstream model. Kept in config for traceability. |
| `copilot-grok-4.7` | Dead upstream model. Kept in config for traceability. |

Both have `model_info.discoverable: false`. They are not returned by `/v1/models` and are rejected by the virtual key with HTTP `403`.

## Routing and reliability

`config.yaml` sets:

```yaml
router_settings:
  num_retries: 2
  timeout: 300
  allowed_fails: 3
  cooldown_time: 30
  fallbacks:
    - copilot-gpt-6.2-sol:
        - claude-copilot-opus-5.5
        - grok-4.7
    - copilot-mai-code-2-flash:
        - grok-4.20-non-reasoning
    - copilot-gpt-5.3-codex:
        - copilot-gpt-6-luna
```

Meaning:

- `num_retries: 2`: retry a failed deployment twice before fallback.
- `timeout: 300`: per-request timeout in seconds.
- `allowed_fails: 3`: failures before a deployment is considered unhealthy.
- `cooldown_time: 30`: seconds before an unhealthy deployment is retried.
- Fallbacks use real model names. No aliases were added.

Known tradeoff: fallback from one Responses API provider to another can drop encrypted reasoning fields. This is accepted to keep coding sessions alive.

## Authentication model

### Master key

- Source: `LITELLM_MASTER_KEY` in `.env`.
- Scope: admin routes, key management, credential management, Admin UI.
- Never store it in OpenCode.

### Virtual key

- Alias: `Opencode`.
- Stored in OpenCode at `~/.local/share/opencode/auth.json` under the `litellm` provider.
- Length: 25 characters. Master key length: 51 characters. Do not confuse them.
- Current limits: `models` = 15 active IDs, `rpm_limit=60`, `tpm_limit=200000`, no expiry, no budget cap.
- Current LiteLLM user: `default_user_id`.

Inspect it safely with the master key:

```bash
set -a; . ./.env; set +a
rtk curl -s -G --data-urlencode "key=$VKEY" \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  http://127.0.0.1:4000/key/info
```

Set `VKEY` from the OpenCode auth store without printing it:

```bash
export VKEY=$(python3 -c "import json; print(json.load(open('/home/overlag/.local/share/opencode/auth.json'))['litellm']['key'])")
```

Update the allowlist or limits with `/key/update`. Use the 15 model IDs from the catalog. Keep the same `user_id` so Copilot per-user connections still apply:

```bash
rtk curl -s -X POST \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"key":"'"$VKEY"'","models":["copilot-gpt-5.3-codex","copilot-gpt-6-luna","copilot-gpt-6.1-terra","copilot-gpt-6.2-astra","copilot-gpt-6.2-sol","copilot-mai-code-2","copilot-mai-code-2-flash","claude-copilot-haiku-5.5","claude-copilot-opus-5.5","claude-copilot-sonnet-5.5","copilot-kimi-k3","grok-4.20","grok-4.20-non-reasoning","grok-4.3","grok-4.7"],"rpm_limit":60,"tpm_limit":200000}' \
  http://127.0.0.1:4000/key/update
```

## xAI OAuth login

The gateway uses `use_xai_oauth: true` on every `xai/...` deployment. No `XAI_API_KEY` is needed.

Token location inside the container:

```text
/root/.config/litellm/xai_oauth/auth.json
```

On the host this is `./xai_oauth/auth.json`, mounted by Compose. Directory mode is `0700`; file mode is `0600`.

This image does not expose `litellm xai-oauth login`. Use the authenticator directly.

The callback server binds to `127.0.0.1` inside the process. A browser on the host cannot complete the loopback callback from a normal Compose container: the published port accepts the TCP connection but the HTTP request is reset before the callback handler sees it. Run the login in a temporary container with host networking, writing the token to the same host directory mounted by Compose.

Stop the gateway first so host port `56121` is free:

```bash
docker compose stop litellm
```

Run the login from the repository root. Keep the process open while you authorize in the browser:

```bash
docker run --rm -it --network host --entrypoint python \
  -v "$PWD/xai_oauth:/root/.config/litellm/xai_oauth" \
  docker.litellm.ai/berriai/litellm:main-stable \
  -c 'from litellm.llms.xai.oauth import XAIOAuthAuthenticator; XAIOAuthAuthenticator().login(force=True)'
```

A browser opens to the xAI OIDC flow. The callback URL is:

```text
http://127.0.0.1:56121/callback
```

If no browser opens, the command prints the URL. Open it manually. Do not close the login process until it reports success.

After success, restart the gateway:

```bash
docker compose start litellm
```

Success looks like:

```text
xAI OAuth login successful. Credentials saved to <auth_file>.
Access token expires at <expires_at>.
```

If xAI shows `Enter this code to finish signing in` / `Copy the code below into Grok Build`, that is xAI's device-code UI. LiteLLM does not read that code. With the same login process still running, open this URL in the local browser using the code from that page and the `state` from the URL printed by that exact login attempt:

```text
http://127.0.0.1:56121/callback?code=<CODE_FROM_PAGE>&state=<STATE_FROM_PRINTED_URL>
```

Codes and states from earlier attempts are invalid. Never paste them into chat.

Notes:

- Access tokens are refreshed automatically from the stored refresh token.
- A `401` after expiry does not trigger automatic replay. Re-run login if that happens.
- Optional env overrides: `XAI_OAUTH_TOKEN_DIR`, `XAI_OAUTH_AUTH_FILE`, `XAI_OAUTH_API_BASE`.
- OAuth uses PKCE with issuer `https://auth.x.ai` and scopes `openid profile email offline_access grok-cli:access api:access`.
- Do not set `XAI_API_KEY`; it skips OAuth.

## GitHub Copilot per-user OAuth login

Credential name used by this repo: `Github Copilot`. Auth type: per-user GitHub OAuth. Provider: GitHub Copilot.

### Admin UI path

1. Open `http://127.0.0.1:4000/ui/`.
2. Go to **Models + Endpoints** → **LLM Credentials** → **Add Credential**.
3. Set:
   - Provider: `GitHub Copilot`
   - Credential Name: `Github Copilot`
   - Auth Type: `Per-user GitHub OAuth`
   - GitHub host: leave empty for `github.com`
4. Save.
5. Go to **LLM Credentials** → **Your connections** → `Github Copilot` → **Connect**.
6. Open `https://github.com/login/device`, enter the shown code, and approve with the account that has the Copilot seat.
7. Status becomes **Connected** with the GitHub login.

For GitHub Enterprise Cloud, use `SUBDOMAIN.ghe.com` as the host. GitHub Enterprise Server is not supported. Changing the host removes existing user connections.

### API alternative

Create the credential with the master key:

```bash
set -a; . ./.env; set +a
rtk curl -s -X POST http://127.0.0.1:4000/credentials \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"credential_name":"Github Copilot","credential_values":{"github_copilot_auth_type":"per_user_oauth"},"credential_info":{"custom_llm_provider":"github_copilot"}}'
```

Start the device flow with the user's virtual key, not the master key:

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

Using the virtual key matters: the connection belongs to the LiteLLM user that key resolves to. Requests must come from that same user. This repo uses `default_user_id`.

`lite credentials connect <name>` appears in newer docs but is not present in this image. Use the UI or the endpoints above.

### Storage and endpoint requirement

- Each user's GitHub access token is encrypted in Postgres.
- No GitHub refresh token is stored.
- A short-lived Copilot token is cached in memory until 60 seconds before expiry.
- GPT and MAI Copilot models require `/v1/responses`. The chat-to-responses bridge loses `litellm_credential_name` and fails with:

```text
litellm.BadRequestError: github_copilot_auth_type per_user_oauth requires litellm_credential_name
```

Do not switch OpenCode to `/v1/chat/completions` for those models.

## OpenCode setup

Config file: `~/.config/opencode/opencode.json`.

Installed OpenCode 1.18.35 uses the legacy V1 shape: singular `provider`, `npm`, and `options`. The V2 `providers` + `settings` shape is ignored with this diagnostic:

```text
configuration compatibility diagnostic path=["providers"] kind=unsupported action="Omitted native setting that cannot be represented in V1"
```

Working shape:

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

`@ai-sdk/openai` is intentional because the gateway uses `/v1/responses`. Do not set `options.apiKey`; the credential comes from the auth store.

Store the virtual key in OpenCode:

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

If OpenCode cached an old credential, restart the process. Then select a model with `/models` or run:

```bash
opencode run -m litellm/grok-4.20-non-reasoning "hello" --print-logs --log-level DEBUG
```

Expected log line:

```text
llm.runtime=ai-sdk llm.provider=litellm llm.model=grok-4.20-non-reasoning
```

## Usage examples

Responses API with the virtual key:

```bash
set -a; . ./.env; set +a
export VKEY=$(python3 -c "import json; print(json.load(open('/home/overlag/.local/share/opencode/auth.json'))['litellm']['key'])")
rtk curl -s -X POST http://127.0.0.1:4000/v1/responses \
  -H "Authorization: Bearer $VKEY" \
  -H 'Content-Type: application/json' \
  -d '{"model":"grok-4.20-non-reasoning","input":"say hi"}'
```

Expected: HTTP `200`, `"status": "completed"`, and text in `output[].content[].text`.

Blocked model check:

```bash
rtk curl -s -o /dev/null -w '%{http_code}\n' -X POST \
  http://127.0.0.1:4000/v1/responses \
  -H "Authorization: Bearer $VKEY" \
  -H 'Content-Type: application/json' \
  -d '{"model":"claude-copilot-fable-5.2","input":"hi"}'
```

Expected: `403`.

List models visible to the virtual key:

```bash
rtk curl -s -H "Authorization: Bearer $VKEY" http://127.0.0.1:4000/v1/models
```

Expected: 15 model IDs. Neither hidden model appears.

## Spend tracking

Spend is written by LiteLLM to `LiteLLM_SpendLogs`. No custom success or failure callbacks are configured.

Query the table:

```bash
set -a; . ./.env; set +a
rtk docker exec -e PGPASSWORD="$POSTGRES_PASSWORD" litellm-db \
  psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" \
  -c 'SELECT count(*) AS total, max("startTime") AS last_start FROM "LiteLLM_SpendLogs";'
```

Per-key spend is also available through `/key/info`.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Health returns `000` after restart | Gateway still starting | Wait ~15s, then `docker logs litellm`. |
| `The requested model '...' is not available for this API key` | Model outside key allowlist | Update `/key/update` with the correct 15 IDs. |
| `github_copilot_auth_type per_user_oauth requires litellm_credential_name` | Chat bridge used for GPT/MAI Copilot | Use `/v1/responses`. Keep `litellm_credential_name` on the deployment. |
| Copilot returns `401` | User connection expired or rejected | Reconnect in Admin UI → **Your connections**. |
| xAI returns `AuthenticationError` | OAuth token missing or expired | Re-run `XAIOAuthAuthenticator().login()` in the container. |
| OpenCode logs V2 `unsupported` diagnostic | V2 config used on 1.18.35 | Use singular `provider` + `npm` + `options`. |
| `No model is available for session` | No credential or model selected | Run `/connect`, then `/models`. |
| Duplicate skill warnings | Same skill installed in multiple dirs | Harmless. Ignore. |

## Known limitations

- GPT/MAI Copilot models do not work through `/v1/chat/completions`.
- Dead models remain in `config.yaml` but are hidden and blocked by the key.
- Cross-provider Responses fallback can drop encrypted reasoning.
- No model aliases. Use real model IDs.
- `rpm_limit=60` and `tpm_limit=200000` are conservative defaults. Tune per workload.
- Guardrails skipped: require an external API key.
- Caching skipped: low hit rate with `previous_response_id`.
- Auto Router skipped: add-on not installed.
- MCP Gateway skipped: OpenCode already has its own MCP servers.
- Admin UI and gateway bind to `127.0.0.1` only.

## Security notes

- Never print `.env`, `auth.json`, master key, virtual key, or OAuth tokens.
- Use `rtk curl` or shell variables so secrets stay out of command output.
- Master key is admin-only. OpenCode gets only the virtual key.
- `.env`, `postgres_data/`, and `xai_oauth/` are gitignored.
- xAI token file is `0600` inside a `0700` directory.
- Keep ports local unless you add TLS and authentication in front.

## References

- LiteLLM virtual keys: https://docs.litellm.ai/docs/proxy/virtual_keys
- LiteLLM load balancing and reliability: https://docs.litellm.ai/docs/proxy/load_balancing
- LiteLLM xAI provider: https://docs.litellm.ai/docs/providers/xai
- LiteLLM GitHub Copilot provider: https://docs.litellm.ai/docs/providers/github_copilot
- LiteLLM Responses API gate: https://github.com/BerriAI/litellm/pull/29747
- LiteLLM Copilot CLI connect: https://github.com/BerriAI/litellm/pull/50863
- OpenCode providers: https://opencode.ai/docs/providers/
- OpenCode CLI providers: https://opencode.ai/docs/cli/providers/
- OpenCode V1 migration notes: https://opencode.ai/docs/migrate-v1/
