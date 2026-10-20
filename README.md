# dep-env-awareness

A Claude Code plugin that audits your project's dependencies, environment configuration, and runtime compatibility — then tells you exactly what's broken and how to fix it.

## What it does

When invoked (or triggered automatically), this skill runs a five-phase diagnostic:

1. **Detect project type** — Identifies ecosystems (Node, Python, Rust, Go, PHP, Ruby, Java) from manifest files
2. **Dependency graph audit** — Maps the full dependency tree, flags duplicates, peer dependency warnings, deprecated and unused packages, lock file drift, and security vulnerabilities
3. **Environment comparison** — Captures runtime versions, compares against version files (`.nvmrc`, `.python-version`, `engines`), diffs `.env.example` vs `.env`, verifies `.env` is gitignored, and checks Docker/container configs for drift
4. **CI/CD pipeline analysis** — Extracts versions and env vars from CI configs, identifies local-vs-CI discrepancies that cause "works on my machine" failures
5. **Structured report** — Outputs findings grouped by severity (critical / warning / recommendation) with exact fix commands and verification steps

## When it triggers

- Dependency conflicts, version mismatches, lock file issues
- "It works on my machine" / CI failures
- Docker or container environment problems
- node_modules issues, virtual environment problems, nvm/pyenv mismatches
- Requests to audit, check, or verify project setup
- Debugging sessions where the root cause appears to be environmental

For Salesforce-specific projects, see the companion [sf-env-awareness](https://github.com/Pierre-louis242/sf-env-awareness) plugin.

## Install

### As a plugin (recommended)

```bash
# From a marketplace (once registered)
/plugin install dep-env-awareness@<marketplace-name>

# Or locally for development/testing
claude --plugin-dir /path/to/dep-env-awareness
```

### As a standalone skill

```bash
git clone https://github.com/Pierre-louis242/dep-env-awareness.git /tmp/dep-env-awareness
cp -r /tmp/dep-env-awareness/skills/dep-env-check ~/.claude/skills/dep-env-check
```

## Usage

The skill triggers automatically when relevant. You can also invoke it directly:

```
# As plugin
/dep-env-awareness:dep-env-check

# As standalone skill
/dep-env-check
```

Or describe what you need naturally:

```
> Why does my build pass locally but fail in GitHub Actions?
> Check my project dependencies for issues
> Audit this monorepo's environment setup
```

## Plugin structure

```
dep-env-awareness/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── dep-env-check/
│       └── SKILL.md
├── README.md
└── LICENSE
```

## Supported ecosystems

| Ecosystem | Manifest | Lock file | Tree command |
|-----------|----------|-----------|--------------|
| Node (npm) | `package.json` | `package-lock.json` | `npm ls --all` |
| Node (yarn) | `package.json` | `yarn.lock` | `yarn list --all` |
| Node (pnpm) | `package.json` | `pnpm-lock.yaml` | `pnpm ls --depth Infinity` |
| Python (pip) | `requirements.txt` | — | `pipdeptree` |
| Python (poetry) | `pyproject.toml` | `poetry.lock` | `poetry show --tree` |
| Rust | `Cargo.toml` | `Cargo.lock` | `cargo tree` |
| Go | `go.mod` | `go.sum` | `go mod graph` |

## Contributing

Issues and PRs welcome. If you'd like to add support for another ecosystem or improve detection heuristics, open a PR with the changes to `skills/dep-env-check/SKILL.md`.

## License

MIT
