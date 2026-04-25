# Vercel Agent Skills Wrapper Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `accelerate-data/vercel-agent-skills` installable as both a Claude and Codex marketplace plugin while preserving its role as a vendored fork of `vercel-labs/agent-skills`.

**Architecture:** Keep upstream skill content as real files in the root `skills/` directory. Add wrapper-owned plugin manifests, agent governance notes, and a scheduled upstream sync workflow that opens reviewable pull requests.

**Tech Stack:** GitHub repository metadata, Claude plugin manifest JSON, Codex plugin manifest JSON, GitHub Actions, shell validation, marketplace `url` sources.

---

## File Structure

- Create: `.claude-plugin/plugin.json`
  - Claude plugin manifest for the wrapper repository.
- Create: `.codex-plugin/plugin.json`
  - Codex plugin manifest for the wrapper repository.
- Modify: `AGENTS.md`
  - Add vendored-content guidance and required warning behavior for AI agents.
- Create: `.github/workflows/sync-upstream.yml`
  - Scheduled/manual workflow that syncs from `vercel-labs/agent-skills` into a PR.
- Modify later in `/Users/hbanerjee/src/plugin-marketplace`: `.claude-plugin/marketplace.json`
  - Point the Claude marketplace entry at `accelerate-data/vercel-agent-skills`.
- Modify later in `/Users/hbanerjee/src/plugin-marketplace`: `.agents/plugins/marketplace.json`
  - Point the Codex marketplace entry at `accelerate-data/vercel-agent-skills`.

## Task 1: Add Claude Plugin Manifest

**Files:**
- Create: `.claude-plugin/plugin.json`

- [ ] **Step 1: Create the manifest directory**

Run:

```bash
mkdir -p .claude-plugin
```

Expected: command exits 0.

- [ ] **Step 2: Add `.claude-plugin/plugin.json`**

Create `.claude-plugin/plugin.json` with:

```json
{
  "name": "vercel-agent-skills",
  "version": "0.1.0",
  "description": "Vendored Vercel agent skills packaged for Accelerate Data Claude marketplace installation.",
  "author": {
    "name": "Accelerate Data",
    "url": "https://github.com/accelerate-data"
  },
  "repository": "https://github.com/accelerate-data/vercel-agent-skills"
}
```

- [ ] **Step 3: Validate JSON**

Run:

```bash
python3 -m json.tool .claude-plugin/plugin.json
```

Expected: pretty-printed JSON and exit 0.

- [ ] **Step 4: Commit**

Run:

```bash
git add .claude-plugin/plugin.json
git commit -m "feat: add Claude plugin manifest"
```

Expected: commit succeeds.

## Task 2: Add Codex Plugin Manifest

**Files:**
- Create: `.codex-plugin/plugin.json`

- [ ] **Step 1: Create the manifest directory**

Run:

```bash
mkdir -p .codex-plugin
```

Expected: command exits 0.

- [ ] **Step 2: Add `.codex-plugin/plugin.json`**

Create `.codex-plugin/plugin.json` with:

```json
{
  "name": "vercel-agent-skills",
  "version": "0.1.0",
  "description": "Vendored Vercel agent skills packaged for Accelerate Data Codex marketplace installation.",
  "author": {
    "name": "Accelerate Data",
    "url": "https://github.com/accelerate-data"
  },
  "repository": "https://github.com/accelerate-data/vercel-agent-skills"
}
```

- [ ] **Step 3: Validate JSON**

Run:

```bash
python3 -m json.tool .codex-plugin/plugin.json
```

Expected: pretty-printed JSON and exit 0.

- [ ] **Step 4: Commit**

Run:

```bash
git add .codex-plugin/plugin.json
git commit -m "feat: add Codex plugin manifest"
```

Expected: commit succeeds.

## Task 3: Add Vendored-Content Agent Guidance

**Files:**
- Modify: `AGENTS.md`

- [ ] **Step 1: Insert vendored repository section near the top of `AGENTS.md`**

Add this section after the opening paragraph:

```markdown
## Vendored Upstream Content

This repository is an Accelerate Data-maintained fork of `vercel-labs/agent-skills`.
The skill content under `skills/` is vendored upstream content.

AI agents may edit wrapper-owned files such as `.claude-plugin/`, `.codex-plugin/`,
`.github/workflows/`, documentation, and validation scripts as normal.

Before editing files under `skills/`, AI agents must warn the user that the file is
upstream-owned and explain that the change may diverge from future upstream syncs.
Proceed only when the user explicitly confirms the local divergence or when the task
is specifically to resolve an upstream sync conflict.
```

- [ ] **Step 2: Review the rendered guidance**

Run:

```bash
sed -n '1,80p' AGENTS.md
```

Expected: the vendored-content section appears before skill-authoring instructions.

- [ ] **Step 3: Commit**

Run:

```bash
git add AGENTS.md
git commit -m "docs: document vendored skill guidance"
```

Expected: commit succeeds.

## Task 4: Add Scheduled Upstream Sync Workflow

**Files:**
- Create: `.github/workflows/sync-upstream.yml`

- [ ] **Step 1: Create workflow directory**

Run:

```bash
mkdir -p .github/workflows
```

Expected: command exits 0.

- [ ] **Step 2: Add workflow**

Create `.github/workflows/sync-upstream.yml` with:

```yaml
name: Sync upstream agent skills

on:
  schedule:
    - cron: "0 9 * * 1"
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  sync:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout fork
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure git identity
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"

      - name: Add upstream remote
        run: git remote add upstream https://github.com/vercel-labs/agent-skills.git

      - name: Fetch upstream
        run: git fetch upstream main

      - name: Create sync branch
        run: git checkout -B sync/upstream-agent-skills

      - name: Merge upstream
        run: git merge upstream/main --no-edit

      - name: Validate wrapper files
        run: |
          python3 -m json.tool .claude-plugin/plugin.json >/dev/null
          python3 -m json.tool .codex-plugin/plugin.json >/dev/null
          test -d skills
          find skills -name SKILL.md -type f | grep -q .

      - name: Push sync branch
        run: git push origin sync/upstream-agent-skills --force-with-lease

      - name: Open sync pull request
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          gh pr view sync/upstream-agent-skills --json number >/dev/null 2>&1 || \
            gh pr create \
              --base main \
              --head sync/upstream-agent-skills \
              --title "Sync upstream vercel-labs/agent-skills" \
              --body-file - <<'BODY'
            Automated sync from `vercel-labs/agent-skills`.

            Review vendored `skills/` changes before merge.
          BODY
```

- [ ] **Step 3: Validate workflow YAML parses**

Run:

```bash
python3 - <<'PY'
from pathlib import Path
import sys

path = Path(".github/workflows/sync-upstream.yml")
text = path.read_text()
for required in [
    "name: Sync upstream agent skills",
    "cron:",
    "https://github.com/vercel-labs/agent-skills.git",
    "git push origin sync/upstream-agent-skills --force-with-lease",
    "gh pr create",
]:
    if required not in text:
        raise SystemExit(f"missing required workflow text: {required}")
print("workflow text checks passed")
PY
```

Expected: `workflow text checks passed`.

- [ ] **Step 4: Commit**

Run:

```bash
git add .github/workflows/sync-upstream.yml
git commit -m "ci: add upstream sync workflow"
```

Expected: commit succeeds.

## Task 5: Smoke Test Plugin Shape

**Files:**
- Read: `.claude-plugin/plugin.json`
- Read: `.codex-plugin/plugin.json`
- Read: `skills/*/SKILL.md`

- [ ] **Step 1: Run local validation checks**

Run:

```bash
python3 -m json.tool .claude-plugin/plugin.json >/dev/null
python3 -m json.tool .codex-plugin/plugin.json >/dev/null
test -d skills
find skills -name SKILL.md -type f | grep -q .
```

Expected: command exits 0.

- [ ] **Step 2: Inspect changed files**

Run:

```bash
git status --short
```

Expected: no uncommitted changes after previous task commits.

## Task 6: Update Plugin Marketplace

**Files:**
- Modify: `/Users/hbanerjee/src/plugin-marketplace/.claude-plugin/marketplace.json`
- Modify: `/Users/hbanerjee/src/plugin-marketplace/.agents/plugins/marketplace.json`
- Modify if needed: `/Users/hbanerjee/src/plugin-marketplace/.claude-plugin/plugin.json`

- [ ] **Step 1: Replace Claude marketplace entry**

In `/Users/hbanerjee/src/plugin-marketplace/.claude-plugin/marketplace.json`, replace the old Vercel entry with:

```json
{
  "name": "vercel-agent-skills",
  "description": "Vendored Vercel agent skills packaged for Accelerate Data Claude and Codex marketplace installation.",
  "strict": false,
  "source": {
    "source": "url",
    "url": "https://github.com/accelerate-data/vercel-agent-skills.git"
  }
}
```

- [ ] **Step 2: Replace Codex marketplace entry**

In `/Users/hbanerjee/src/plugin-marketplace/.agents/plugins/marketplace.json`, replace the old Vercel entry with:

```json
{
  "name": "vercel-agent-skills",
  "source": {
    "source": "url",
    "url": "https://github.com/accelerate-data/vercel-agent-skills.git"
  },
  "policy": {
    "installation": "AVAILABLE",
    "authentication": "ON_INSTALL"
  },
  "category": "Coding"
}
```

- [ ] **Step 3: Keep marketplace versions aligned**

If `.claude-plugin/marketplace.json` `metadata.version` changes, update
`/Users/hbanerjee/src/plugin-marketplace/.claude-plugin/plugin.json` to the same version.

- [ ] **Step 4: Validate marketplace**

Run from `/Users/hbanerjee/src/plugin-marketplace`:

```bash
python3 scripts/validate-marketplace.py --base-ref HEAD
python3 -m unittest tests/test_validate_marketplace.py
codex plugin marketplace add .
```

Expected:

```text
marketplace validation passed
Ran 21 tests
OK
Marketplace `ad-internal-marketplace` is already added
```

- [ ] **Step 5: Commit marketplace update**

Run from `/Users/hbanerjee/src/plugin-marketplace`:

```bash
git add .claude-plugin/marketplace.json .agents/plugins/marketplace.json .claude-plugin/plugin.json
git commit -m "feat: add Vercel agent skills marketplace entry"
```

Expected: commit succeeds.

## Self-Review

- Spec coverage: The plan covers plugin manifests, vendored-agent warning notes,
  upstream sync workflow, validation, and marketplace source updates.
- Placeholder scan: No placeholder markers or unspecified implementation steps remain.
- Scope check: The plan is one cohesive wrapper-and-marketplace task. Upstream
  skill behavior changes are intentionally out of scope.
