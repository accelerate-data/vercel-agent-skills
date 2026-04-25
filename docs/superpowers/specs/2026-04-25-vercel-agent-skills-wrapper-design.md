# Vercel Agent Skills Wrapper Design

## Context

`accelerate-data/vercel-agent-skills` is an Accelerate Data-owned fork of
`vercel-labs/agent-skills`. The upstream repository is the source of truth for
the Vercel-authored skills, but it does not provide the plugin manifests needed
for reliable installation through the Accelerate Data Claude and Codex
marketplaces.

Claude Code can consume the upstream repository as a loose skill collection when
the marketplace entry uses `strict: false`. Codex needs a real plugin shape, so
the marketplace should point at this fork after it contains the required plugin
metadata.

## Goals

- Publish the Vercel skill collection through the Accelerate Data marketplace as
  `vercel-agent-skills`.
- Keep the root `skills/` directory as real files from upstream, not symlinks.
- Add Claude and Codex plugin manifests without changing upstream skill content.
- Make the vendored nature of the repository explicit to AI agents and humans.
- Require agents to warn before editing vendored upstream skill content.
- Keep future upstream syncs simple and reviewable.

## Non-Goals

- Rewrite or repackage the upstream Vercel skills.
- Add local Accelerate Data-specific skill behavior.
- Build a custom sync service.
- Automatically merge upstream changes without review.

## Repository Shape

The fork keeps the upstream layout intact:

```text
vercel-agent-skills/
  .claude-plugin/plugin.json
  .codex-plugin/plugin.json
  AGENTS.md
  README.md
  skills/
    composition-patterns/
    deploy-to-vercel/
    react-best-practices/
    react-native-skills/
    react-view-transitions/
    vercel-cli-with-tokens/
    web-design-guidelines/
  .github/workflows/sync-upstream.yml
```

The `skills/` directory remains a real root directory because that is the
lowest-risk shape for marketplace installers. Symlinks are avoided because some
runtime installers, archive extractors, or scanners may skip or normalize them.

## Marketplace Identity

The plugin name is `vercel-agent-skills`, not `typescript-best-practices`.
That name makes clear that the entry is the full Vercel skill collection and
that the repository is a vendored wrapper maintained for team use.

Both marketplaces should use a whole-repo source:

```json
{
  "source": "url",
  "url": "https://github.com/accelerate-data/vercel-agent-skills.git"
}
```

The Claude marketplace entry may keep `strict: false` if needed for compatibility
with loose skill-collection conventions. The Codex marketplace entry should use
the same source URL after `.codex-plugin/plugin.json` exists.

## Agent Safety Note

`AGENTS.md` should state that this repository vendors upstream content from
`vercel-labs/agent-skills`. AI agents may edit wrapper files such as plugin
manifests, documentation, sync workflow files, and validation scripts as normal.

Before editing vendored skill content under `skills/`, agents must warn the user
that the file is upstream-owned and explain the sync risk. They may proceed only
when the user explicitly confirms the local divergence or when the task is
specifically to resolve an upstream sync conflict.

## Upstream Sync

A scheduled GitHub Actions workflow should fetch `vercel-labs/agent-skills`,
merge the upstream default branch into a sync branch, run validation, and open a
pull request. The workflow should not push directly to `main`.

The sync PR gives the team a review point for upstream skill changes and for any
conflicts against local wrapper files.

## Validation

The implementation should verify:

- `.claude-plugin/plugin.json` is valid JSON.
- `.codex-plugin/plugin.json` is valid JSON.
- `skills/` exists and contains `SKILL.md` files.
- GitHub Actions workflow syntax is parseable YAML.
- The plugin can be referenced from `plugin-marketplace` as a `url` source.

Marketplace changes belong in `plugin-marketplace`, not in this fork.
