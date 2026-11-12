# AGENTS.md

## What this repo is

A single [Agent Skill](https://agentskills.io/specification) that teaches DeepSeek Harness (DSH) plugin
development. There is no build, no tests, and no package manifest — the deliverable is Markdown.

## Layout and conventions

- `SKILL.md` — the router. Frontmatter holds `name`/`description`; the body only routes to `references/`.
  Keep it short and add a row to the task-routing table instead of inlining new material.
- `references/NN-*.md` — one self-contained guide per topic, flat (no nested reference chains).
  `NN` is a two-digit prefix that sets reading order.
- `README.md` — human-facing overview of the skill, not loaded by agents.

## Hard rules for the content

- Every reference starts with a `> **实时文档（先读，可能与本文不同步）**` block listing live URLs.
  This is a product requirement: when the skill and the docs disagree, the docs win.
- Prose is Chinese; code, identifiers, paths, and API names stay in English.
- Doc URLs follow the site's bilingual scheme: Chinese pages under `/develop/...` and `/reference/...`,
  English pages the same path prefixed with `/en`.

## Facts worth remembering

- Installation is via `npx skills add axetroy/dsh-plugin-skills`. The CLI discovers a skill at the
  repository root when `SKILL.md` sits there — no `skills/` subdirectory is needed.
- Because the CLI clones the whole repo, keep `AGENTS.md` and `README.md` here; they are installed
  alongside the skill but are not loaded by agents.
- Upstream docs site: <https://deepseek-harness.github.io/deepseek-harness/> (VitePress, bilingual).
  The site source is `docs/` in <https://github.com/deepseek-ai/deepseek-harness>; the rendered site
  has no sitemap, so enumerate pages by following `href="/deepseek-harness/..."` links.
- Read upstream Markdown from the repo (shallow sparse clone of `docs/`) rather than scraping HTML —
  it is far cleaner and preserves code fences.
- Reference implementations to cite: `packages/shell/tool-bash` (tools),
  `packages/llm/llm-deepseek` and `packages/llm/llm-pi-ai` (LLM adapters).
- The upstream repo ships its own skills under `.agents/skills/`; match their `description` style
  (task-trigger phrasing, not a summary).

## Validate before finishing

```sh
pip install skills-ref
python3 -c "from pathlib import Path; from skills_ref.validator import validate; print(validate(Path('.')))"
```

Run it with the repo root as the path, or it reports a spurious directory-name mismatch.
`errors: []` means the frontmatter, naming rules, and required fields are all valid.

Then confirm every URL in the skill still resolves:

```sh
grep -rho 'https://[^ )"<>]*' SKILL.md README.md references | sort -u |
  xargs -I{} sh -c 'curl -s -o /dev/null -w "%{http_code} {}\n" -L --max-time 25 {}' | grep -v '^200'
```

Expect no output. A dead link means the docs moved; fix the URL and, if the content changed,
update the affected reference too.
