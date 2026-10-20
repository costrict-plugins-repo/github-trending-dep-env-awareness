---
name: dep-env-check
description: >
  Audit and diagnose project dependencies, environment configuration, and runtime compatibility issues.
  Use this skill whenever the user mentions dependency conflicts, version mismatches, environment issues,
  Docker problems, CI failures, "it works on my machine", broken builds after updates, lock file confusion,
  missing packages, incompatible versions, node_modules issues, virtual environment problems, nvm, pyenv,
  or asks to check/audit/verify their project setup. Also trigger when the user asks "why does this fail
  in CI but not locally", "what versions am I running", or wants to understand their dependency tree.
  Even if not explicitly asked, use this when debugging suggests an environment or dependency root cause.
  For Salesforce-specific projects, use the sf-env-check skill instead.
---

# Dependency & Environment Awareness

A structured skill for auditing, diagnosing, and resolving dependency and environment issues across projects.

## Scope Control

If the user asks to check only a specific aspect (e.g., "just check my lock file", "are my env vars ok?", "audit security only"), run **only** the relevant phase(s) and skip the rest. Map common requests to phases:

- Lock file / install issues → Phase 2b
- Security / vulnerabilities → Phase 2c
- Version mismatch / runtime → Phase 3a
- Env vars / secrets → Phase 3b
- Docker → Phase 3c
- CI vs local → Phase 4
- Full audit → All phases

Only run the full pipeline when the user asks for a general audit or when the problem is unclear.

---

## Phase 0: Quick Triage Mode

If the user's immediate question is "why is my build broken?" or "why does install fail?", skip the full audit and run a fast triage:

1. **Check the error output** — ask for or read the failing command's output
2. **Lock file sync** — `npm ci --dry-run` / `pip install -r requirements.txt --dry-run` to surface version conflicts immediately
3. **Runtime mismatch** — compare `.nvmrc` / `.python-version` / `engines` against active runtime
4. **Missing env vars** — diff `.env.example` vs `.env`
5. **Last known change** — `git diff HEAD~1 -- package.json requirements.txt Cargo.toml` to see if deps just changed

Present the likely root cause and fix. If the triage is inconclusive, fall through to the full audit below.

---

## Phase 1: Detect Project Type

Identify the project ecosystem(s) by checking for manifest files:

```
package.json, package-lock.json, yarn.lock, pnpm-lock.yaml  → Node.js
requirements.txt, Pipfile, pyproject.toml, poetry.lock, uv.lock  → Python
Cargo.toml, Cargo.lock  → Rust
go.mod, go.sum  → Go
composer.json  → PHP
Gemfile, Gemfile.lock  → Ruby
build.gradle, pom.xml  → Java/Kotlin
```

If multiple ecosystems are present (monorepo), note each and audit separately.

## Phase 2: Dependency Graph Audit

### 2a. Tree & Duplicates

Run the appropriate command to get the full dependency tree:

| Ecosystem | Command |
|-----------|---------|
| Node (npm) | `npm ls --all 2>&1` |
| Node (yarn) | `yarn list --all` |
| Node (pnpm) | `pnpm ls --depth Infinity` |
| Python (pip) | `pip list --format=json && pipdeptree --warn fail` |
| Python (poetry) | `poetry show --tree` |
| Rust | `cargo tree` |
| Go | `go mod graph` |

Look for:
- **Duplicate packages** at different versions (especially in Node)
- **Peer dependency warnings** — these cause silent runtime failures
- **Deprecated packages** — `npm outdated` / `pip list --outdated`
- **Unused dependencies** — cross-reference imports in source code vs declared deps

### 2b. Lock File Health

- Verify lock file exists and is committed to git (`git ls-files <lockfile>`)
- Check lock file is in sync with manifest: `npm ci --dry-run` or equivalent
- Flag if `.gitignore` excludes the lock file (it shouldn't for applications)
- Compare lock file age vs manifest age: `git log -1 --format=%ci <lockfile>` vs `git log -1 --format=%ci <manifest>`

### 2c. Security Scan

- Node: `npm audit --json`
- Python: `pip-audit` or `safety check`
- Rust: `cargo audit`
- Go: `govulncheck ./...`

Report critical/high vulnerabilities with remediation paths.

## Phase 3: Environment Comparison

### 3a. Runtime Versions

Capture and compare:

```bash
# Language runtimes
node --version 2>/dev/null
python3 --version 2>/dev/null
go version 2>/dev/null
rustc --version 2>/dev/null
java --version 2>/dev/null

# Package managers
npm --version 2>/dev/null
pip --version 2>/dev/null
cargo --version 2>/dev/null

# Version managers
nvm --version 2>/dev/null || echo "nvm not installed"
pyenv --version 2>/dev/null || echo "pyenv not installed"
```

Cross-reference with:
- `.nvmrc`, `.node-version`, `.python-version`, `.tool-versions` (asdf)
- `engines` field in `package.json`
- `python_requires` in `pyproject.toml`
- CI config files for expected versions

Flag mismatches between version manager config and active runtime version (e.g. `.nvmrc` says 18 but `node --version` reports 20).

### 3b. Environment Variables

- Check `.env`, `.env.example`, `.env.local` files
- Diff `.env.example` vs actual `.env` to find missing vars
- **Verify `.env` is in `.gitignore`** — if it's tracked or unignored, flag as a **critical security issue** (secrets may be committed to history)
- Check for `.env` files already committed: `git log --all --diff-filter=A -- '.env' '.env.local' '.env.production'`
- Check CI config (`.github/workflows/*.yml`, `.gitlab-ci.yml`, `Jenkinsfile`) for env vars
- Flag secrets that appear hardcoded in source (basic grep for API_KEY, SECRET, TOKEN, PASSWORD patterns)

### 3c. Docker / Container Analysis

If `Dockerfile` or `docker-compose.yml` exists:

- Check base image versions (flag `latest` tags — these cause drift)
- Compare installed packages in container vs local
- Verify volume mounts match expected paths
- Check multi-stage build efficiency
- Validate `COPY` commands include necessary files (especially lock files before `npm install`)

## Phase 4: CI/CD Pipeline Analysis

If CI config exists:

- Extract the runtime versions, environment variables, and install commands
- Compare CI environment setup with local environment
- Flag differences that could cause "works locally, fails in CI":
  - Different Node/Python versions
  - Missing env vars
  - Different OS (e.g., macOS local vs Ubuntu CI)
  - Caching issues (`node_modules` cached with wrong Node version)
  - Network restrictions (private registries, VPN-only packages)

## Phase 5: Report & Remediation

Present findings as a structured report:

### Summary
One-paragraph health assessment.

### Critical Issues (fix now)
Issues that will cause build/runtime failures.

### Warnings (fix soon)
Drift, deprecations, security issues.

### Recommendations (nice to have)
Cleanup, optimization, best practices.

For each issue, provide:
1. **What**: Clear description of the problem
2. **Why it matters**: Impact on builds, runtime, security
3. **Fix**: Exact command(s) to resolve it
4. **Verify**: How to confirm the fix worked

## Edge Cases

- **Monorepos**: Audit each workspace/package separately, then check cross-workspace version consistency
- **Private registries**: Note if `.npmrc` or `pip.conf` references private registries that may not be accessible in all environments
- **Platform-specific deps**: Flag `optionalDependencies` or platform-conditional installs that differ between macOS/Linux/Windows
