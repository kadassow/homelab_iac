# NutriTrace AI (Trace AI) — Meal Photo Setup Options & Decision Notes

Documents the two viable paths for enabling NutriTrace's built-in "Trace AI"
meal-photo feature (`propose_food` photo flow — attach a photo, get an
editable nutrition card back). Meant to sit alongside
`AUTHENTIK_OIDC_SETUP.md`, `KOPIA_BACKUP_STRATEGY.md`, and
`PROXMOX_GPU_SETUP.md` — not built yet, capturing the plan and reasoning so
it isn't lost before this gets picked up.

## Background

NutriTrace has this feature natively — no custom glue code, webhook, or
companion service required. It's controlled entirely by env vars on the
`nutritrace` service:

- `AI_PROVIDER` — locks Trace to one of `claude` | `openai` | `gemini` |
  `oai-compat`. Unset means users configure their own key per-browser in
  Settings (not desirable here — want this centrally configured for family
  members, not something each person sets up individually).
- `AI_BASE_URL` — only used with `AI_PROVIDER=oai-compat`. Points at any
  OpenAI-compatible `/v1/chat/completions` endpoint (Ollama, LM Studio,
  LocalAI). Requests are proxied **through the NutriTrace server**, not
  called from the browser — meaning a local Ollama container only needs to
  be reachable on the same Docker network as `nutritrace`, no NPM/Authentik
  exposure needed.
- Cloud providers (`claude`/`openai`/`gemini`) need a plain API key env var
  for that provider (naming per provider — confirm exact var name against
  NutriTrace's `docs/self-hosting/env-vars/` page at implementation time,
  since this wasn't pinned down precisely during planning).

Both paths slot into the exact same pattern already used for
`NUTRITRACE_OIDC_*` — new `vault_`-prefixed secret (if any) in
`group_vars/all/vault.yml`, plain reference in `group_vars/all/vars.yml`,
rendered into `.env` via `templates/mediastack.env.j2`, consumed by
`docker/compose/vm2_services/mediastack/docker-compose.yml`.

## Option A — Local model via Ollama (`oai-compat`)

**Pros:** Nothing leaves the network, matches NutriTrace's own "no cloud
sync unless you opt in" design and the broader homelab self-hosting
philosophy. No ongoing cost, no API key/secret to manage.

**Cons:** Hardware is an N150 with no discrete GPU — CPU-only inference
(iGPU is earmarked for Jellyfin QuickSync, see `PROXMOX_GPU_SETUP.md`, and
isn't a good fit for LLM inference workloads anyway). Expect several
seconds per photo, not instant. Accuracy from a small (~2–3B) local vision
model will be noticeably rougher than a frontier cloud model — broad
strokes right (rice, chicken, broccoli), portion sizes and less-common
items less reliable.

### Model choice

| Model | Size | Notes |
|---|---|---|
| **Moondream2** | ~1.9B | Purpose-built for captioning/VQA, lightest CPU footprint — best starting point |
| **Qwen2.5-VL:3b** | ~3B | Step up in quality, slower on CPU — try if Moondream's output feels too thin |

Both are pullable directly via Ollama (`ollama pull moondream`,
`ollama pull qwen2.5vl:3b`).

### Setup steps (not yet built)

1. **Add an `ollama` service** to `mediastack`'s compose file, on the
   existing `media-network` bridge (same network `nutritrace` is already
   on):
   ```yaml
   ollama:
     image: ollama/ollama:latest
     container_name: ollama
     restart: unless-stopped
     volumes:
       - $LOCALDOCKER_DIR/ollama:/root/.ollama
     networks:
       - media-network
   ```
   Model weights go on `LOCALDOCKER_DIR` (local disk), **not** CIFS — not a
   SQLite-locking concern like the rest of the local-disk rule, just that a
   multi-GB model file should have fast local reads rather than getting
   pulled over SMB on every container restart.

2. **Pull the model post-deploy.** Either a one-off `docker exec ollama
   ollama pull moondream`, or a proper Ansible task (`community.docker.docker_container_exec`
   or a plain `command:` against the running container) so it's
   reproducible on a fresh deploy rather than a manual step that silently
   needs redoing.

3. **Wire the env vars.** In `group_vars/all/vars.yml`:
   ```yaml
   NUTRITRACE_AI_PROVIDER: "oai-compat"
   NUTRITRACE_AI_BASE_URL: "http://ollama:11434"
   ```
   No vault entry needed — no API key for a local model. Add both to
   `templates/mediastack.env.j2` and the `nutritrace` service's
   `environment:` block in `mediastack/docker-compose.yml`, same pattern as
   the existing `NUTRITRACE_OIDC_*` vars.

4. **Verify.** Confirm `docker exec nutritrace curl http://ollama:11434`
   resolves from inside the NutriTrace container (network reachability),
   then test the actual `propose_food` photo flow in the app.

## Option B — Claude API (`AI_PROVIDER=claude`)

**Pros:** Meaningfully better accuracy/portion-size judgment than a small
local model. Trivial cost at real-world usage volumes (a household logging
meal photos a few times a day). No local compute burden on the N150 at all.

**Cons:** Meal photos leave the network per-call — a deliberate exception to
carve out, not a silent default, given the rest of this homelab is built to
avoid that.

### Cost reality check

Billed per token, not a flat subscription — separate from claude.ai's free
chat tier, which has no API access. Roughly (check
`docs.anthropic.com/en/api/pricing` before committing — this drifts):

| Model | Input / 1M tokens | Output / 1M tokens | Notes |
|---|---|---|---|
| Claude Haiku 4.5 | ~$1 | ~$5 | Cheapest, still vision-capable — fine for "what's roughly on this plate" |
| Claude Sonnet 5 | ~$2 | ~$10 | Better reasoning if portion/ingredient accuracy matters more |

A single meal photo is a few hundred to ~1500 tokens depending on
resolution, plus a small prompt/tool-use overhead. At household-scale usage
this is cents per day — not a "need Pro/Max" situation, just a normal API
key with a few dollars of prepaid credit.

### Setup steps (not yet built)

1. **Generate an API key** at `console.anthropic.com`. New accounts get a
   small one-time credit; after that it's pay-as-you-go, card required.

2. **Store it in the vault**, same pattern as `vault_nutritrace_oidc_client_secret`:
   ```yaml
   # group_vars/all/vault.yml (encrypted)
   vault_nutritrace_ai_api_key:
   ```
   Reference it plainly in `group_vars/all/vars.yml`:
   ```yaml
   NUTRITRACE_AI_PROVIDER: "claude"
   NUTRITRACE_AI_API_KEY: "{{ vault_nutritrace_ai_api_key }}"
   ```
   (Confirm the exact expected var name — `AI_API_KEY` vs a
   provider-specific name like `ANTHROPIC_API_KEY` — against NutriTrace's
   env-var docs at implementation time; this wasn't pinned down precisely
   during planning.)

3. **Wire into `.env` and compose**, same as every other
   `NUTRITRACE_*` var today — add to `templates/mediastack.env.j2` and the
   `nutritrace` service's `environment:` block, `no_log: true` on the
   render task since it's secret-bearing.

4. **Verify.** Test the `propose_food` photo flow in the app; confirm no
   plaintext key shows up in `docker inspect nutritrace` output or
   container logs.

## Which to pick — not decided yet

Both are roughly equal effort to wire up (same env-var pattern either way).
The real decision is philosophical, not technical: local keeps the
"nothing leaves the network" property NutriTrace and this homelab are both
built around, at the cost of speed and accuracy; Claude API costs pennies
and gives materially better results, at the cost of that one exception.

One option not yet explored: whether `AI_PROVIDER` can be swapped later
without any data loss/migration — if so, there's no real risk in starting
with Option A (local, free, matches the privacy-first pattern) and
revisiting Option B later if the accuracy gap actually bothers anyone using
it day to day, rather than committing to a cloud dependency up front.

## Status

- ⬜ Not started — this doc reflects planning only, nothing deployed yet.
- ⬜ Exact env var names (`AI_API_KEY` vs provider-specific, precise
  `oai-compat` var spelling) need confirming against NutriTrace's
  `docs/self-hosting/env-vars/` page before writing the actual Ansible
  tasks/templates — noted above wherever this doc is making an assumption
  rather than a confirmed fact.

## Open questions / future considerations

- Does switching `AI_PROVIDER` later require re-doing any linked
  data/history, or is it a clean swap? Affects how much weight to put on
  "just start with local and revisit."
- If Option A is chosen, is Moondream2's accuracy actually good enough in
  practice, or does it need bumping to Qwen2.5-VL:3b (slower, better)? Only
  answerable by testing once deployed.
- Worth revisiting whether the N150's iGPU could assist local inference via
  OpenVINO instead of pure CPU (there's an OpenVINO int4-quantized
  Qwen2-VL-2B build aimed at exactly this) — secondary optimization, not a
  blocker for a first pass on CPU.
