# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Agent Skills for Context Engineering is an open collection of Agent Skills teaching context engineering and harness engineering principles for production AI agent systems. Skills are platform-agnostic (Claude Code, Cursor, Codex/OpenAI Agent Skills, GitHub Copilot, any Open Plugins-conformant tool). v2.6.0 adds the self-managed-context skill (model-controlled live context editing: context as an editable file, self-editing harness invariants, edit-position cache cost, budget nudges, and steering or training context-editing strategy) on top of the long-horizon-prompting skill (v2.5.0), the self-improvement-loops skill (v2.4.0), and the file-based researcher operating system with deterministic gates, cross-platform Agent Skills validation, and a continuous loop. Use the [generated live inventory](researcher/generated/corpus-summary.md) for current corpus counts.

Context engineering is the discipline of curating everything that enters a model's context window (system prompts, tool definitions, retrieved documents, message history, tool outputs) to maximize signal within limited attention budget.

## Repository Structure

- `skills/` - published skill directories, each containing a `SKILL.md` with YAML frontmatter (`name`, `description`) and optional `references/` and `scripts/` subdirectories
- `examples/` - complete demonstration projects; the generated inventory is the count authority
- `docs/` - Research materials and reference documentation
- `researcher/` - File-based research-to-skill operating system: rubrics, mechanism registry, claim provenance, corpus index, run state machine, adversarial benchmarks, continuous loop, launchd service definitions
- `template/SKILL.md` - Canonical skill template (use when creating new skills)
- `SKILL.md` (root) - Collection-level metadata and skill map
- `.claude-plugin/marketplace.json` - Claude Code marketplace manifest (single bundled plugin, v2.6.0)
- `.plugin/plugin.json` - Open Plugins format manifest (v2.6.0)

## Setup

```
python3 -m pip install -r requirements-dev.txt   # pyyaml, skills-ref, jsonschema, referencing
```

CI uses Python 3.12 and Node 22. Python 3.13 also works locally. No top-level build system; repo-level gates and per-project tooling below.

## Build & Test Commands

### Fast pre-commit gate set (all run in CI on every push to main and every PR)

All of these finish in seconds, except the unit tests (about 10 s). Run them from the repo root:

```
python3 -m unittest discover -s researcher/scripts/tests -p 'test_*.py'   # all researcher unit tests
python3 researcher/scripts/validate_governance.py --check     # authority model and generated view
python3 researcher/scripts/build_inventory.py --check         # corpus references and generated inventory
python3 researcher/scripts/validate_export.py check --staging-dir researcher/exports/examples/restricted-citation-v1  # public export boundary
python3 researcher/scripts/validate_schemas.py --check        # Python schema contract
python3 researcher/scripts/migrate_legacy.py --dry-run        # artifact contract migration
python3 researcher/scripts/validate_platform_compat.py --require-reference-validator # Agent Skills reference validator + Cursor/Claude/Codex install-layout simulation
python3 researcher/scripts/validate_repo.py --strict          # corpus structure, manifests, rubric math, mechanism registry, claims, corpus index, activation cases, benchmark scenarios, run artifacts
python3 researcher/scripts/skill_health.py --strict --no-history  # deterministic skill-body quality gate
python3 researcher/scripts/check_activation_cases.py          # skill-boundary regression fixtures
python3 researcher/scripts/run_benchmarks.py                  # adversarial benchmark harness + repo + activation gates
```

When a `--check` gate fails because generated files are stale, regenerate them with the same script and `--write` (`build_inventory.py`, `validate_governance.py`, `validate_schemas.py`) and commit the regenerated files under `researcher/corpus/`, `researcher/generated/`, `governance/generated/`, or `researcher/schemas/generated/`. Never hand-edit generated files.

CI also runs `python -m py_compile` on every script in `researcher/scripts/`. When you add a new script or test, add it to the compile list in `.github/workflows/validate.yml`.

### TypeScript checks in CI

```
cd researcher/schemas/typescript && npm ci --ignore-scripts && npm run typecheck && npm test
cd researcher/benchmarks/sdk-runner && npm ci --ignore-scripts && npm run typecheck
npm run router:dry-run -- --models gpt-5.5 --reps 1 --max-runs 118 --max-budget-usd 2        # from sdk-runner/
npm run effectiveness:dry-run -- --models gpt-5.5 --reps 1 --max-runs 12 --max-budget-usd 2  # from sdk-runner/
```

Dry runs make no SDK calls. `router:run` and `effectiveness:run` call a paid API: never run them without explicit user approval and cost gates (see AGENTS.md).

If you add a router benchmark prompt or a skill, the router `--max-runs 118` cap in CI may need raising.

### Long-horizon prompt lab (examples/long-horizon-prompt-lab)

```
python3 examples/long-horizon-prompt-lab/scripts/build_lab.py
python3 examples/long-horizon-prompt-lab/scripts/validate_site.py
```

CI fails if the build changes `data/prompt-pairs.json`, `ui/data.js`, or `ui/prompts/`. After editing lab sources or the `long-horizon-prompting` rubric, rebuild and commit the outputs. `npm run serve` serves the site on port 8000. `.github/workflows/deploy-prompt-lab.yml` deploys it.

### Per-run readiness (active runs only)

```
python3 researcher/scripts/validate_run.py --run-dir researcher/runs/<run-id>
```

### Continuous loop (manual or launchd)

```
python3 researcher/scripts/loop_discover.py
python3 researcher/scripts/loop_step.py --allow-fetch
python3 researcher/scripts/loop_daily.py
python3 researcher/scripts/loop_status.py

researcher/orchestration/launchd/install.sh    # macOS daemon
researcher/orchestration/launchd/uninstall.sh
```

### Example projects (not in CI)

#### examples/llm-as-judge-skills (TypeScript, Node >= 18)
```
cd examples/llm-as-judge-skills
npm install
npm run build        # tsc
npm test             # vitest
npm run lint         # eslint
npm run format       # prettier
npm run typecheck    # tsc --noEmit
```

#### examples/interleaved-thinking (Python >= 3.10)
```
cd examples/interleaved-thinking
pip install -e ".[dev]"
pytest               # pytest + pytest-asyncio
ruff check .         # linting (100 char line length)
```

#### examples/digital-brain-skill (Node.js)
```
cd examples/digital-brain-skill
npm run setup
npm run weekly-review
npm run content-ideas
npm run stale-contacts
```

## Code Style

- **Python scripts in `researcher/scripts/`**: stdlib first (only the packages in `requirements-dev.txt`), runnable as `python3 researcher/scripts/<name>.py` from the repo root. Tests use `unittest`, live in `researcher/scripts/tests/test_*.py`, and must not need network access or API keys.
- **Shared files**: atomic writes (`tempfile` + `os.replace`) and `fcntl` locks for anything the loop touches.
- **JSONL ledgers and registries** (`registry.jsonl`, `claims/index.jsonl`, `ledgers/*.jsonl`, `fixtures/activation-cases.jsonl`): append-only, one JSON object per line. Never rewrite or reorder past entries.
- **YAML frontmatter**: quote any `description` that contains a colon, or strict parsers in Cursor, Claude Code, and Codex reject the skill.
- **Heuristics**: no regex or keyword-list heuristics in skills or scripts. Use mechanism-level criteria, rubrics, and evidence-backed validation.
- **Prose (skills, docs, README, CHANGELOG, PR bodies)**: technical CTO tone, direct and precise, trade-offs stated upfront. No marketing language, exclamation marks, emojis, or em dashes.
- **CHANGELOG.md**: add every change under `## [Unreleased]`, grouped by `### Added` / `### Changed`, and list the corpus wiring it touched (mechanisms, claims, activation cases, regenerated files).
- **Release version** lives in three places that must match: `.claude-plugin/marketplace.json`, `.plugin/plugin.json`, and root `SKILL.md`.

## Skill Authoring Rules

When creating or editing skills:

1. **SKILL.md must stay under 500 lines**: move detailed content to `references/` directory
2. **YAML frontmatter is required**: must include `name` and `description` fields
3. **Folder naming**: lowercase with hyphens (e.g., `context-fundamentals`)
4. **Write in third person**: descriptions are injected into system prompts; inconsistent POV causes discovery issues
5. **Platform-agnostic**: no vendor-locked examples or platform-specific tool names without abstraction
6. **Token-conscious**: challenge each paragraph and assume an advanced audience
7. **Body standard**: include `When to Activate`, `Core Concepts`, `Practical Guidance`, `Examples`, `Guidelines`, `Gotchas`, `Integration`, and `References`
8. **Explicit boundaries**: every `When to Activate` section needs positive triggers plus a `Do not activate` block routing adjacent work to the right skill
9. **Include a Gotchas section**: experience-derived failure modes are the highest-signal content in any skill
10. **Update root README.md** when adding new skills
11. **Update marketplace/plugin manifests** when adding skills (`.claude-plugin/marketplace.json`, `.plugin/plugin.json`)
12. **Update the corpus index** (`researcher/corpus/index.json`) to map the new skill to activation scenarios, mechanism IDs, and claim IDs
13. **Update mechanisms and claims**: add registry entries for reusable behavior changes and `claim-*` provenance for numeric, benchmark, volatile, or vendor-performance claims
14. **Run `validate_platform_compat.py --require-reference-validator`, `validate_repo.py --strict`, `skill_health.py --strict --no-history`, `check_activation_cases.py`, and `run_benchmarks.py`** before committing skill changes

## Researcher OS Rules

When working through the researcher operating system:

1. **Initialize runs via `research_loop.py init`**: it creates `run-state.json`, queue entry, thread log, source evaluation scaffold, and mechanism proposal template
2. **Advance state explicitly**: use `retrieve`, `evaluate`, `propose`, `novelty`, `validate-run`, `pr-ready`, `close` subcommands; do not edit `run-state.json` by hand
3. **Promote mechanisms only after run readiness**: `research_loop.py promote-mechanisms` requires `--reviewed-by` and a passing run-readiness check
4. **Add claim provenance** to `researcher/claims/index.jsonl` for any numeric, benchmark, or volatile claim added to a skill
5. **Never invoke paid LLMs from the continuous loop**: HTTP retrieval is stdlib-only, judge adapters are explicitly out of scope until budget-gated
6. **Never commit runtime queue/report files**: `.gitignore` covers `researcher/queue/*.jsonl`, `researcher/reports/{logs,snapshots,loop-events.jsonl,loop-failures.jsonl,status.md,parked-review.md}`, and `researcher/runs/*/` except the seed run

## Plugin Architecture

All published skills are distributed as a single plugin (`context-engineering`) in the marketplace manifest. This avoids cache duplication: Claude Code caches each plugin's `source` directory separately, so multiple plugins pointing to `source: "./"` would each cache a full copy of the repo.

Progressive disclosure pattern: only skill names/descriptions load at startup; full content loads on activation.

## Git and PR Workflow

- Work on a feature branch, never directly on `main`. CI validates every PR, including stacked PRs whose base is another proposal branch.
- Run the fast gate set above before every commit that touches skills, `researcher/`, `governance/`, or manifests.
- Never push, open a PR, or merge without explicit user approval for that specific action. Approval for one push does not cover the next.
- Changes to `docs/specs/SPEC-000-program-constitution.md` or `governance/constitution.yaml` use the `.github/PULL_REQUEST_TEMPLATE/constitution-amendment.md` template.
- Do not commit anything under `Private/`, `dashboard/`, `.cursor/`, or the runtime paths listed in `.gitignore`.

## Workspace Memory

`AGENTS.md` holds learned user preferences and workspace facts (benchmark stages, Cursor SDK cost rules, ledger event shape, API key handling). Read it before benchmark, ledger, or release work. Architecture decisions live in `docs/decisions/` and specs in `docs/specs/`.

## Key Design Principles

- **Context quality over quantity**: attention scarcity and lost-in-middle behavior mean more context is not always better
- **Sub-agents isolate context**: they exist to manage attention budget, not simulate org roles
- **Skills reference each other**: use plain text skill names (not links) in Integration sections to avoid cross-directory reference issues
- **Examples use Python pseudocode**: conceptual demonstrations that work across environments, not production-ready implementations
- **Deterministic first, model-judged second**: structure, schema, rubric math, manifest sync, retrieval status, and registry shape must pass before any LLM judge is invoked
- **Human-controlled merge**: agents may prepare PRs and pass gates, but push and merge always require explicit human approval
