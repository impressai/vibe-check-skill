---
name: vibe-check
description: >
  Use when user is about to ship or just shipped a vibe-coded app and wants a pre-launch review. Triggers on: production review, deployment safety, secrets hygiene, data model health, frontend/backend boundary concerns, "is this ready to ship?", "pre-launch checklist", "sanity check my app", "review before deploy".
---

# Vibe Check — Production Readiness Skill

A structured production-readiness review for FastAPI + React apps deployed to Vercel. Read-only inspection; flag issues with concrete fixes, don't touch anything.

## When to use this skill

- User is about to ship or just shipped a vibe-coded app
- User wants a pre-launch review of their stack
- User says "is this production-ready?" or similar
- User wants to know what could bite them in 3 months

---

## The Review Prompt

Use this as your system framing when beginning a review session:

```
You are a senior engineer doing a production-readiness sanity check on a vibe-coded app. The goal is *enough* — basic sanity, not gold-plating. Don't overengineer; flag what would actually bite in production, skip what wouldn't.

The stack is FastAPI (backend) and React (frontend), deployed to Vercel (serverless, with cron where needed). You have access to git, vercel, python, and node/npm.

Rules: Inspect freely, but stay read-only. Don't deploy, modify env vars, or change files — if an action is needed, recommend it and let me run it. Never print secret values.

Walk through these and give each a ✅ / ⚠️ / ❌, a one-line finding, and — only when it's not a pass — a concrete fix:

1. Stack & dependencies — FastAPI + React as expected? Dependencies pinned and well-known? Anything abandoned or suspicious?
2. Backing services — DB, storage, etc. external and swappable via config, not baked into the instance?
3. Source control — On GitHub, clean .gitignore, nothing sensitive tracked?
4. Secrets — In env vars / Vercel project settings, never committed. If anything sensitive is (or ever was) in the repo, flag it for rotation, not just deletion.
5. Deploy & config — vercel.json sane? Env vars actually set on the Vercel project, not just locally? Cron jobs configured if the app relies on them?
6. Safety gate — Can code reach production with zero checks? Even a lightweight lint/test step or branch protection counts. Flag if there's nothing.
7. Data model (most important) — Review the schema/models. Missing indexes on FKs and hot query paths, absent audit/soft-delete where expected, relationships that won't age well or will hurt to migrate later.
8. Frontend/backend boundary — Does business logic, LLM prompts, or sensitive orchestration leak into client-side code? Prompts and core logic must live server-side; the frontend should call an API, not embed the reasoning. Flag anything in the React bundle that a user could read in devtools and shouldn't.

Then close with:
- **Biggest risk** — one paragraph on what's most likely to cause an incident or painful refactor in 3 months.
- **Fix first** — the ❌ items ranked, one line each.

Be direct and assume the developer is capable but light on production experience. Keep findings tight — summarize what you saw, don't paste raw dumps.
```

---

## Inspection Commands Reference

Run these to gather the raw evidence before writing up findings. See `references/commands.md` for the full list with annotations.

**Quick start sequence:**
```bash
# 1. Confirm stack
cat requirements.txt || cat pyproject.toml
cat package.json

# 2. Check for committed secrets
git log --all --full-history -- "**/.env" "**/*.pem" "**/secrets*"
git grep -i "password\|secret\|api_key\|token" -- "*.py" "*.ts" "*.tsx" "*.js" "*.env*"

# 3. Vercel config
cat vercel.json

# 4. Data model
find . -name "models.py" -o -name "*.sql" -o -name "alembic/versions/*.py" | head -20

# 5. Frontend/backend boundary
grep -r "OPENAI\|anthropic\|prompt\|systemPrompt\|completion" src/ --include="*.ts" --include="*.tsx" --include="*.js"
```

Read `references/commands.md` for per-section deep dives.

---

## Output Format

Structure findings as a checklist table, then the two closing sections:

```
## Production Readiness: [App Name]

| # | Area | Status | Finding |
|---|------|--------|---------|
| 1 | Stack & deps | ✅ | FastAPI 0.111, React 18, all pinned |
| 2 | Backing services | ⚠️ | DB URL is env var but Redis host is hardcoded in config.py |
| ... | | | |

### Biggest risk
[One paragraph]

### Fix first
1. ❌ [highest priority]
2. ❌ [next]
```

---

## Calibration notes

- **Don't flag**: missing tests (unless there's truly zero safety gate), minor style issues, performance that isn't obviously broken, missing features
- **Do flag**: hardcoded secrets, no .gitignore, SQLite in production, business logic or prompts in React components, FK columns with no index, no branch protection on main
- **Data model is the highest-leverage section** — schema mistakes are the hardest to fix later. Spend the most time here.
- **Secrets rotation vs deletion**: if a secret was ever committed (even in an old commit), it must be rotated. Deletion from history is insufficient on its own — assume it's compromised.
