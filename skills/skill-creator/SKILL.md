---
name: skill-creator
description: Create cecli skills from Memorizer facts and user input.
version: 0.2.0
author: Tom (tomjuggler), ox-alpha
license: MIT
platforms: [linux, macos, windows]
tags: [skill-authoring, memorizer, knowledge-capture]
---

# Create Skill

Turn accumulated project knowledge — Memorizer facts, user-supplied
information, and optional documentation research — into a well-formed
cecli skill (SKILL.md). Distill, never dump: raw memories become
checkable procedures.

## When to Use

- User asks to "create a skill", "make a skill from what we learned",
  or "turn this into a skill"
- A workflow has repeated across 2+ sessions and should be captured
- Worked example: "create a new skill with what we have learned about
  Android development so far, look up the documentation first then
  create the skill" → topic=Android dev learnings, research=YES,
  scope=default (project-local)

Don't use for: importing existing Hermes skills (use `hermes-import`);
editing an existing skill (patch its SKILL.md directly).

## Procedure

### Step 1 — Parse the request

Extract before gathering anything:
- **Topic/domain** (e.g. "Android development")
- **Research flag** — phrases like "look up the documentation first"
  or "research X" enable Step 3; otherwise skip it
- **Scope** — project-local `.cecli/skills/` (default) or global
  `~/.cecli/skills/`; ask only if genuinely ambiguous. Custom `skills_paths`
  in agent-config add further search dirs (priority: project first, then
  configured order, home last)
- **Name hint** — kebab-case, ≤64 chars; derive from topic if absent

Done when: topic, research flag, scope, and candidate name are decided.

### Step 2 — Gather knowledge (three sources)

Collect from all three; do not stop at the first non-empty source.

a. **Memorizer** — delegate to the `memorizer` sub-agent (async=false):
   ```
   Search the Fact database for everything about <TOPIC>.
   Probe keywords: <topic terms>. Probe tags: preferences, structures,
   goals, decisions, relationships, entities, plus domain tags.
   Return matching facts grouped by tag with full contents and any
   dates, or the single word EMPTY. Read-only: insert/delete nothing.
   ```
b. **Project artifacts** — `.cecli/skills/*/references/*.md`, README.md,
   `git log --oneline -30` for hard-won lessons.
c. **User-supplied info** — constraints and must-includes stated in the
   request override gathered facts on conflict.

Done when: a knowledge inventory exists with every item tagged
fact / artifact / user / research-later — or EMPTY noted per source.

### Step 3 — Optional research (only if flagged)

- Library/framework docs: Context7 `resolve-library-id`, then
  `query-docs` (max 3 calls per question)
- Specific pages: `fetch`
- Record source URLs; they belong in the skill body or references/

Done when: each open question has either a cited answer or is dropped.

### Step 4 — Synthesize

- Keep durable, reusable knowledge; cut one-off task state
- Dedupe: `ls` both skill scopes; extend an existing skill instead of
  creating a near-duplicate sibling. Same name in both scopes → project
  copy wins (shadowing); reserve that for deliberate overrides
- Plan size: ~100 lines simple, ~200 max; overflow goes into
  `references/*.md` beside SKILL.md
- Flag secrets/IPs/credentials found in facts — confirm with the user
  before baking them in

Done when: an outline exists with a source annotation per section.

### Step 5 — Author SKILL.md

Frontmatter must start at byte 0 (no leading blank line/BOM):

```yaml
---
name: <kebab-name>
description: <capability, <=60 chars, one sentence, ends with period>
license: MIT
# allowed-tools: [tool-names] # optional; restricts tools while active
metadata:
  version: 0.1.0
  author: <human>, ox-alpha
  tags: [short, tags]
---
```

Body order: `# Title` + 2-3 sentence intro → `## When to Use` (+
"Don't use for") → `## Prerequisites` → `## Procedure` (numbered steps,
each ending in a checkable completion criterion) → `## Pitfalls` →
`## Verification`.

Optional subfolders beside SKILL.md, loaded by category: `references/`
(`**/*.md` docs), `scripts/` (executables), `assets/` (binaries/config),
`evals/` (`evals.json` tests). Only `name` + `description` frontmatter are
enforced by the loader — everything else is convention.

Rules: capability statements over narration; no "be careful" filler;
strong leading words; distill facts into steps — never paste raw memory.

### Step 6 — Validate

```bash
python3 -c '
import re, yaml, pathlib
c = pathlib.Path("<scope>/<name>/SKILL.md").read_text()
assert c.startswith("---"), "must start with --- at byte 0"
m = re.search(r"\n---\s*\n", c[3:])
fm = yaml.safe_load(c[3:m.start()+3])
assert "name" in fm and "description" in fm
d = fm["description"]
assert len(d) <= 60 and d.endswith("."), f"description {len(d)} chars"
print("OK:", fm["name"])
'
```

Done when: the snippet prints OK.

### Step 7 — Install & report

Write to `<scope>/<name>/SKILL.md`. Report: skill name, path, one-line
capability, and source counts (memorizer facts / artifacts / user items /
research citations). New skills appear next session by default; for
immediate pickup enable `hot_reload: true` in agent-config (per-turn
rescan) or run `/hot-reload` (restarts program, preserves session state).

## Pitfalls

- **Empty fact DB** — common (verified: 0 facts on first
  query); fall back to artifacts + user input rather than stalling
- **Stale facts** — prefer dated entries; carry dates into the skill
  ("verified YYYY-MM")
- **Secrets in memories** — Things like IPs and domain names are fine; API keys are not;
  confirm before inclusion
- **Router skills** — a skill that only points at other skills is churn;
  the catalog already routes
- **Same-session loading** — discovery caches at session start; without
  `hot_reload: true` or a `/hot-reload`, freshly written skills stay
  invisible until next session

## Verification

- [ ] Validation snippet passes (frontmatter parses; description ≤60 + ".")
- [ ] Every Procedure step ends in a checkable criterion
- [ ] Knowledge inventory accounted for, EMPTY sources documented
- [ ] No duplication with existing skills in either scope
- [ ] File at `<scope>/<name>/SKILL.md`; report delivered to user