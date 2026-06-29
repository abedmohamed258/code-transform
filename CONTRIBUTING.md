# Contributing to OmniProject AI (code-transform)

Thanks for investing your time in contributing! This document describes the development workflow for the OmniProject AI / code-transform skill.

## Quick Start

```bash
# Clone
git clone https://github.com/<your-username>/code-transform.git
cd code-transform

# Verify skill is discoverable
npx skills add . --list

# Install in Claude Code
npx skills add . -a claude-code -y

# Optional: install Playwright for browser features
pip install playwright
playwright install chromium
```

## Development Workflow

1. **Create a branch**:
   ```bash
   git checkout -b feature/your-feature
   # OR
   git checkout -b fix/your-fix
   git checkout -b docs/your-docs-update
   ```

2. **Make changes** under the appropriate directory:
   - `SKILL.md` — main orchestrator (only for workflow/phase changes)
   - `skills/<sub-skill-name>/SKILL.md` — sub-skill specifications
   - `scripts/` — Python and Bash scripts
   - `references/` — deep-dive reference docs
   - `assets/` — templates
   - `evals/cases/` — eval test cases

3. **Validate** (see Validation Checklist below)

4. **Commit** using Conventional Commits (see Commit Format)

5. **Open a PR** with a clear description of what changed and why

## Validation Checklist

Before opening a PR, verify:

- [ ] `name` field in `SKILL.md` matches the directory name (`code-transform`)
- [ ] `description` field is < 1024 characters
- [ ] All scripts in `scripts/` are executable (`chmod +x scripts/*.sh scripts/*.py`)
- [ ] All Python scripts pass `python3 -m py_compile`
- [ ] All Bash scripts pass `bash -n` (syntax check)
- [ ] No broken references (every `references/NN-*.md` mentioned in `SKILL.md` exists)
- [ ] Every sub-skill in `skills/*/SKILL.md` has ≥100 lines (no stubs)
- [ ] Version numbers in `SKILL.md`, `README.md`, `AGENTS.md` all match
- [ ] "Scripts (N)" count in `SKILL.md` matches the actual table row count
- [ ] "Sub-Skill Routing (N sub-skills)" count matches actual `skills/*/` directory count
- [ ] `npx skills add . --list` discovers the skill without errors

## Testing

```bash
# Syntax check all scripts
for f in scripts/*.sh; do bash -n "$f" && echo "OK: $f"; done
for f in scripts/*.py; do python3 -m py_compile "$f" && echo "OK: $f"; done

# Help text for every script (smoke test)
for f in scripts/*.sh; do bash "$f" --help 2>&1 | head -5; done
for f in scripts/*.py; do python3 "$f" --help 2>&1 | head -5; done

# Run eval harness (requires test cases under evals/cases/)
python3 scripts/run_eval.py --cases evals/cases --output evals/benchmark.json
python3 scripts/generate_review.py evals/benchmark.json --open

# Validate Sprint-1-M1 integrity (recovery check)
bash scripts/validate_sprint1.sh

# Run unit tests (if tests/ directory exists)
python3 -m pytest tests/ -v
```

## Commit Format

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

[optional body]

[optional footer]
Closes #<issue>
```

### Types

| Type | When to use |
|------|-------------|
| `feat` | New feature or sub-skill |
| `fix` | Bug fix |
| `docs` | Documentation only (README, CHANGELOG, references) |
| `refactor` | Code restructuring without behavior change |
| `perf` | Performance improvement |
| `test` | Adding or updating tests / eval cases |
| `chore` | Maintenance (deps, configs, CI) |
| `security` | Security fix |
| `design` | Intentional visual/UI baseline change (for visual-regression commits) |
| `fix(a11y)` | Accessibility fix (auto-applied by accessibility-auditor) |

### Scopes

Common scopes: `auth`, `browser`, `a11y`, `visual`, `spec`, `phase4`, `phase6`, `phase9`, `eval`, `docs`, `script`.

### Examples

```
feat(browser): add browser-launcher sub-skill with full lifecycle spec

fix(a11y): auto-add alt text to images missing it [SP-42]

docs: sync README/AGENTS/CHANGELOG to v18.2.0

test(eval): add fix-keyboard-trap case for accessibility coverage

design(login): update button color from blue to green (intentional)
```

## Sub-Skill Contribution Guidelines

When adding a new sub-skill:

1. **Pick a name** that matches the routing trigger (e.g. `auth-setup` for auth-related work)
2. **Create `skills/<name>/SKILL.md`** with this structure:
   ```markdown
   ---
   name: <name>
   description: "<when to use this sub-skill>"
   license: MIT
   metadata:
     version: "1.0"
     author: OmniProject AI
     category: <category>
   ---

   # <Title>

   ## When to Use
   ## What It Does
   ## Integration Contract (INPUT/OUTPUT schema)
   ## CLI
   ## Decision Tree
   ## Failure Modes & Recovery
   ## Self-Healing Loop
   ## Quality Gates
   ## Tools
   ## Hard Rules
   ```
3. **Minimum 150 lines** — no stubs. If you can't fill 150 lines, the sub-skill is too narrow; consider merging with another.
4. **Add to `SKILL.md` routing table** under the appropriate phase
5. **Add to `AGENTS.md` mirror** under the same phase
6. **Update counts**: sub-skill count in `SKILL.md` heading, `AGENTS.md`, `README.md`

## Eval Case Contribution Guidelines

When adding a new eval case under `evals/cases/<case-name>/`:

1. **`case.json`** with this schema:
   ```json
   {
     "name": "<case-name>",
     "description": "<one-line summary>",
     "difficulty": "easy|medium|hard",
     "category": "security|accessibility|visual-regression|code-smell|tests|...",
     "prompt": "<what the agent is asked to do>",
     "assertions": [
       {"type": "file_contains", "file": "src/...", "pattern": "..."},
       {"type": "file_not_contains", "file": "src/...", "pattern": "..."}
     ]
   }
   ```
2. **`src/`** — the buggy/initial state
3. **`solution/src/`** — the correct final state (used for assertion validation)
4. **Assertion types**: `file_contains`, `file_not_contains`, `no_secrets`, `a11y_violations_zero`, `visual_diff_below_tolerance`, `file_exists`
5. **Update `evals/trigger_queries_v2.json`** if the case introduces a new trigger pattern

## Cross-Platform Notes

### NTFS Execution Bit Loss

If you're contributing from Windows (NTFS), Git may lose the executable bit on `scripts/*.sh` and `scripts/*.py`. Fix:

```bash
git update-index --chmod=+x scripts/my-script.sh
git commit -m "fix: restore executable bit on my-script.sh"
```

The `.gitattributes` file should already enforce this for new files, but existing files may need manual fixing.

### Line Endings

`.gitattributes` enforces `* text=auto` with explicit `eol=lf` for `.sh`, `.py`, `.md`. Do not commit CRLF line endings.

## CI Pipeline

PRs trigger `.github/workflows/validate.yml` which:

1. Validates skill is discoverable via `npx skills add . --list`
2. Runs `python3 -m py_compile` on every Python script
3. Runs `bash -n` on every Bash script
4. Checks every sub-skill directory has a `SKILL.md` ≥100 lines
5. Verifies version numbers in SKILL.md / README.md / AGENTS.md match
6. Runs the eval harness (`run_eval.py`) and reports pass rate
7. Runs `validate_sprint1.sh` (recovery integrity)

**PR is blocked if any check fails.**

## Release Process (for maintainers)

1. Bump version in `SKILL.md` (metadata.version), `README.md` (badge), `AGENTS.md` (header line)
2. Add a `## [X.Y.Z] — YYYY-MM-DD` section to `CHANGELOG.md` with all changes
3. Update `RELEASE_NOTES_v<X>.md` (or create a new one for major versions)
4. Commit: `release: vX.Y.Z`
5. Tag: `git tag -a Production-Candidate-v<X.Y.Z> -m "Release vX.Y.Z"`
6. Push tags: `git push --tags`
7. Build ZIP: `zip -r code-transform-v<X.Y.Z>.zip . -x ".git/*" -x "*.pyc" -x "__pycache__/*"`
8. Verify ZIP: `unzip -l code-transform-v<X.Y.Z>.zip | wc -l` (sanity check file count)
9. Attach ZIP to GitHub Release

## Code of Conduct

- Be kind. Disagreements are fine; personal attacks are not.
- Document your reasoning. "Why" matters more than "what" in a spec-driven project.
- If you're unsure, ask. Opening a draft PR for discussion is welcome.
- Respect the Iron Law: never leave the codebase in a broken state.

## License

By contributing, you agree your contributions are licensed under the MIT license (see `LICENSE.txt`).
