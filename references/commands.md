# Inspection Commands Reference

Per-section commands for the vibe-check production review. All read-only.

---

## 1. Stack & Dependencies

```bash
# Python
cat requirements.txt
cat pyproject.toml 2>/dev/null
pip list --outdated 2>/dev/null | head -20

# Node
cat package.json
cat package-lock.json | python3 -c "import json,sys; d=json.load(sys.stdin); print(list(d.get('packages',{}).keys())[:30])" 2>/dev/null

# Check for unpinned deps (ranges like ^, ~, *)
grep -E '[\^~\*]' requirements.txt
grep -E '[\^~\*]' package.json

# Any abandoned/suspicious packages
npm audit --audit-level=high 2>/dev/null | head -40
pip-audit 2>/dev/null | head -40
```

**What to look for:** unpinned versions, packages last updated 2+ years ago, anything with <1k weekly downloads that isn't yours.

---

## 2. Backing Services

```bash
# Find DB/storage config
grep -rE "sqlite|localhost|127\.0\.0\.|/tmp/" --include="*.py" --include="*.env*" . | grep -v ".pyc" | grep -v __pycache__

# Check if connection strings use env vars
grep -rE "DATABASE_URL|REDIS_URL|S3_BUCKET|STORAGE" --include="*.py" .

# Look for hardcoded hosts
grep -rE '(host|url|endpoint)\s*=\s*"[^$]' --include="*.py" . | grep -v "test\|#"
```

**What to look for:** SQLite in production, hardcoded IPs or hostnames, file storage paths on the instance, anything that ties the app to a specific server.

---

## 3. Source Control

```bash
# Check .gitignore covers the basics
cat .gitignore

# Look for sensitive files actually tracked
git ls-files | grep -iE "\.env|\.pem|\.key|secret|credential|password"

# Remote
git remote -v
```

**What to look for:** `.env` files tracked, private keys, `.gitignore` missing `*.env`, `__pycache__/`, `.vercel/`, `node_modules/`.

---

## 4. Secrets

```bash
# Scan current files
git grep -iE "password\s*=|api_key\s*=|secret\s*=|token\s*=" -- "*.py" "*.ts" "*.tsx" "*.js" "*.json" | grep -v "test\|spec\|example\|placeholder\|your_"

# Scan all history (slow on large repos — add --max-count=100 if needed)
git log --all -p --follow -- "**/.env" | grep "^+" | grep -iE "key|secret|password|token" | head -30

# Check .env exists but isn't tracked
ls -la .env* 2>/dev/null
git status .env* 2>/dev/null
```

**Key rule:** if a secret appears in any commit, treat it as compromised. Removing from history is not enough — rotate it.

---

## 5. Deploy & Config

```bash
# Vercel config
cat vercel.json 2>/dev/null

# List env vars set on Vercel project (names only, not values)
vercel env ls 2>/dev/null

# Check if local .env vars match what's on Vercel
diff <(grep -oP '^[A-Z_]+(?==)' .env 2>/dev/null | sort) <(vercel env ls 2>/dev/null | awk 'NR>2{print $1}' | sort) 2>/dev/null

# Cron config
grep -A5 "crons" vercel.json 2>/dev/null
```

**What to look for:** env vars defined locally but missing from Vercel project settings (silent failures on deploy), cron routes that exist in code but aren't registered in vercel.json, rewrites that expose internals.

---

## 6. Safety Gate

```bash
# GitHub Actions
ls -la .github/workflows/ 2>/dev/null
cat .github/workflows/*.yml 2>/dev/null | grep -E "on:|push:|pull_request:|lint|test|check" | head -20

# Pre-commit hooks
cat .pre-commit-config.yaml 2>/dev/null
ls .git/hooks/ 2>/dev/null

# Branch protection (requires gh CLI)
gh api repos/:owner/:repo/branches/main/protection 2>/dev/null | python3 -m json.tool | head -30
```

**Minimum bar:** at least one of — GitHub Actions running on PR, branch protection requiring review, pre-commit hooks with lint. Zero checks = ❌.

---

## 7. Data Model

```bash
# Find models
find . -name "models.py" | xargs cat 2>/dev/null
find . -path "*/alembic/versions/*.py" | sort | tail -5 | xargs cat 2>/dev/null
find . -name "*.sql" | xargs cat 2>/dev/null

# Check for explicit indexes
grep -n "Index\|index=True\|__table_args__" models.py 2>/dev/null

# Check FK columns (should all have indexes)
grep -n "ForeignKey\|relationship" models.py 2>/dev/null

# Soft delete / audit fields
grep -n "deleted_at\|is_deleted\|updated_at\|created_at" models.py 2>/dev/null
```

**What to look for:**
- FK columns with no `index=True` or explicit `Index()` — these will cause slow JOINs at scale
- No `created_at`/`updated_at` on core tables — painful to add later
- Missing `deleted_at` on tables where users would expect undo/recovery
- `String` columns where `Enum` or `Integer` FK would be safer
- Any relationship that looks like it'll become M2M but is modeled as one-to-many

---

## 8. Frontend/Backend Boundary

```bash
# Look for prompts, API keys, or business logic in React code
grep -rn "OPENAI\|ANTHROPIC\|CLAUDE\|GPT" src/ --include="*.ts" --include="*.tsx" --include="*.js" 2>/dev/null
grep -rn "systemPrompt\|userPrompt\|prompt\s*=" src/ --include="*.ts" --include="*.tsx" --include="*.js" 2>/dev/null | grep -v "// "

# Check for API keys directly referenced in frontend
grep -rn "process\.env\." src/ --include="*.ts" --include="*.tsx" | grep -viE "REACT_APP_|VITE_|NEXT_PUBLIC_|API_URL"

# Look for fetch calls going to LLM APIs directly (not your backend)
grep -rn "api\.openai\|api\.anthropic\|generativelanguage\.googleapis" src/ --include="*.ts" --include="*.tsx" --include="*.js" 2>/dev/null

# Check bundle for anything that shouldn't be public (rough heuristic)
ls -lh dist/ build/ .next/ 2>/dev/null | head -10
```

**What to look for:**
- LLM API calls made directly from React (bypassing backend)
- Prompt strings embedded in `.tsx` files — a user can read these in devtools
- `process.env.OPENAI_API_KEY` referenced in frontend code
- Authorization or pricing logic computed client-side
- Any env var in frontend that isn't explicitly public (VITE_/REACT_APP_ prefix)
