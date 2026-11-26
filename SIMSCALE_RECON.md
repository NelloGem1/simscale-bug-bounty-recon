# OP-SIM-1 — SimScale Bug Bounty Recon Report

**Program:** SimScale Public Bug Bounty (Intigriti) — €250–€6,000
**Researcher:** nellogem@intigriti.me (accounts: NelloGem uidExt 1682399096, NelloGem1 uidExt 90987717)
**Date:** 2026-11-26
**Result:** No reportable vulnerability found. Full authz held under two-account testing.

## Attack surface mapped

### api.simscale.com (HAL+JSON, Spring/Java)
- 136 endpoints enumerated via `/v1/openapi.json` (see `simscale_openapi.json`)
- Auth: `X-API-KEY` header, scoped keys (read / project:full tested)
- projectId format: 19-digit snowflakes (unguessable), spaceId: UUIDv4

### www.simscale.com/api (dashboard SPA API)
Discovered via `dash.js` bundle analysis. Session auth via `authtoken` cookie.
Key endpoint families:
- `/api/v1/engineering-ai/*` — chats, agents, model-catalog (Claude Sonnet backend)
- `/api/v1/projects/{owner}/{slug}/*` — results, topicid, copy
- `/api/v2/permissions/projects/{id}` — ACL read (POST grant)
- `/api/v2/projects/*` — simulations, count-runs, shared-with-me
- `/api/v3/projects/*` — trash/restore/purge lifecycle
- `/api/v2/physics-ai/projects/*` — training pipelines (pro feature)
- `/api/v1/spaces/*`, `/api/v1/users/{name}/settingsmap`, `/api/v2/users/{uid}/balances`
- `/api/v1/oauth/grants`, `/api/v1/invites`, `/api/v1/analytics/{id}/signed-url`

### Auth flow (reverse-engineered)
1. `POST /api/v1/auth/login` → `{uid, sid}`
2. `POST ws-long.simscale.com/api/v1/auth/session-sync {uid, sid}` → workbench session
3. `authtoken` cookie (base64 `uid:secret`) authorizes `www.simscale.com/api/*`

## Tests executed (all properly denied)

| Test | Result |
|---|---|
| PUT foreign public project (rename) | 403 Insufficient permission |
| POST simulation on foreign project | 403 Insufficient permission |
| POST perms grant on foreign project (acct B) | 403 |
| PUT space / POST folder in foreign space (acct B) | 403 |
| Copy foreign project into foreign spaceId | 403 |
| Cross-account chat GET (B → A's chat) | 403 "User does not own this chat" |
| settingsmap / balances / invites (B → A) | 403 |
| v3 trash/{foreignId} | 403 |
| Malformed HTML uploaded as STEP CAD | rejected invalid-file |
| S3 temp bucket listing | AccessDenied |
| Private project creation (Community plan) | plan-denied |

## Notable observations (info-level, not reportable)
- Java class names leak in 400 errors (`com.simscale.workflows...`)
- `/api/v1/projects/{owner}/{slug}/results` exposes run metadata for public projects (by design)
- Engineering-AI chat accepts arbitrary `project_id` binding (public projects only reachable)
- `private-project-creation-denied` — private projects are paid-tier only

## Untested / next steps
- physics-ai training pipeline (requires pro feature access)
- CAD/mesh parser fuzzing (real campaign, not API probing)
- Race conditions on copy/permission operations
- Monitors/task_lists in engineering-ai chats
- Organization-scoped endpoints (needs org account)

## Files
- `simscale_openapi.json` — full API spec (136 endpoints)
- `dash.js` — dashboard SPA bundle (endpoint source)
- API keys: NOT committed (live credentials, scoped read/project-full)
