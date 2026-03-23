---
name: launch-team
description: >
  Scaffold and launch a coordinated multi-agent implementation team from a spec
  file. Use when the user wants to implement a spec with parallel specialist
  agents, spin up a team, or configure multi-agent orchestration.
---

# Launch Team

Scaffold and launch a coordinated multi-agent implementation team from a spec file.
A lead agent orchestrates specialist teammates through phased implementation with
plan approval gates, parallel builds, and adversarial review.

**Requires:** Claude Code v2.1.32+, `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` environment variable.

## Critical: Tool Usage

This skill uses the Claude Code Agent Teams feature to launch and coordinate
real agent processes. Two rules apply throughout:

1. **Every user-facing question MUST be asked by calling the AskUserQuestion tool.**
   Do NOT paraphrase questions as plain text, present options in markdown, or ask
   for confirmation via regular messages. If this skill says "ask", "present", or
   "confirm" — that means call the AskUserQuestion tool.

2. **The team MUST be created with TeamCreate before any work begins.**
   After expanding the prompt template, call TeamCreate to set up the team.
   You then become the team lead and follow the expanded template directly —
   do NOT spawn a separate lead agent. The prompt template's TEAM WORKFLOW
   section covers how to spawn teammates, assign tasks, and coordinate.

## Navigation

At any step, if the user selects Other and types "back", "redo", or "start over",
return to the appropriate previous step and re-present that question. This applies
to all interactive configuration steps below.

## Pre-flight

Before any configuration, perform these checks in order:

1. **Exit plan mode** — if the current session is in plan mode, call ExitPlanMode
   before proceeding. Do not begin configuration until plan mode is fully exited.

2. **Resolve spec and focus** — look for a file path in the user's message (they
   may say "implement specs/foo.md", pass `--spec path`, or just mention a
   filename). If no path is found, use the current session plan as the spec. If
   neither exists, use AskUserQuestion to ask the user to provide a spec file path.
   Stop if no spec can be resolved.

   Also extract any **focus instructions** — additional text in the user's message
   beyond the file path that narrows scope (e.g. "focus on the auth module",
   "specifically the webhook handlers", "only phase 2 items"). Store this as
   `{FOCUS}`. If the user's message contains only a file path with no extra
   instructions, `{FOCUS}` is empty.

3. **Read and validate the spec** — attempt to load the resolved spec file. If the
   file does not exist or cannot be read, use AskUserQuestion:
   - **Question**: "Could not read {path}. Provide the correct spec file path?"
   - **Header**: "Spec"
   - **Options**:
     - `Browse` — List markdown files in the current directory to choose from
     - `Retry` — Try the same path again

   If the spec is read successfully but contains fewer than 50 words, use
   AskUserQuestion:
   - **Question**: "This spec is very short ({N} words). It may not contain enough detail for effective team planning. Continue anyway?"
   - **Header**: "Spec"
   - **Options**:
     - `Continue` — Use the spec as-is
     - `Different spec` — Provide a different spec file

## Interactive Configuration

Configure the team through three AskUserQuestion steps. Infer sensible defaults from
the spec content, but always let the user confirm or override.

### Step 1 — Classify spec type

Analyse the spec content and infer which type best matches. Present using AskUserQuestion:

- **Question**: "What type of project does this spec describe?"
- **Header**: "Spec type"
- **Options** (mark the inferred type as recommended):
  - `Web application` — Full-stack web app with frontend and backend
  - `API / backend service` — Server-side API, service, or backend integration
  - `Data pipeline / ML` — Data processing, ML models, or AI pipelines
  - `Other` — Mobile apps, CLI tools, internal platforms, DevOps, or anything else

If the user selects "Other", present a follow-up AskUserQuestion to disambiguate:
- **Question**: "Which best describes this project?"
- **Header**: "Type"
- **Options** (mark the inferred sub-type as recommended):
  - `Infrastructure / DevOps` — CI/CD, IaC, monitoring, deployment
  - `Mobile app` — iOS, Android, or cross-platform mobile
  - `Internal tooling / CLI` — Developer tools, CLIs, shared libraries, internal platforms

Use the full team table below to map the confirmed type to default roles:

| Spec type | Implement | Agile | Adversarial |
|---|---|---|---|
| Web application | backend-engineer, frontend-engineer, database-architect | qa-engineer, technical-writer | architecture-critic, security-auditor, ux-skeptic |
| API / backend service | backend-engineer, database-architect | qa-engineer, researcher, technical-writer | architecture-critic, security-auditor, performance-devil |
| Data pipeline / ML | ml-engineer, backend-engineer, database-architect | qa-engineer, researcher | architecture-critic, performance-devil, security-auditor |
| Infrastructure / DevOps | devops-engineer, platform-engineer | researcher, technical-writer | architecture-critic, security-auditor, performance-devil |
| Mobile app | mobile-engineer, backend-engineer | qa-engineer, technical-writer | ux-skeptic, security-auditor, performance-devil |
| Internal tooling / CLI | backend-engineer, platform-engineer | qa-engineer, technical-writer | architecture-critic, performance-devil |

### Step 2 — Configure team roles

Based on the confirmed spec type, pre-select the default roles from the table above.

Present the default team for confirmation using AskUserQuestion:

- **Question**: "The default team for a {TYPE} project is: {LIST}. Adjust the team?"
- **Header**: "Team"
- **Options**:
  - `Use defaults (Recommended)` — Use the pre-selected team as shown
  - `Add roles` — Add additional specialists to the team
  - `Remove roles` — Drop roles that aren't needed
  - `Custom team` — Specify the exact roles to include

If the user selects "Add roles", present a follow-up AskUserQuestion:
- **Question**: "Which roles to add?"
- **Header**: "Add"
- **Options**: List up to 4 roles NOT already in the default set, picking the most
  relevant based on the spec. The user can also type custom role names via "Other".

If the user selects "Remove roles", present:
- **Question**: "Which roles to remove?"
- **Header**: "Remove"
- **Options**: List up to 4 roles FROM the default set (prioritise the ones least
  relevant to the spec). Removing a role drops it entirely from the team.

If the user selects "Custom team", present:
- **Question**: "Describe your desired team composition"
- **Header**: "Custom"
- **Options**: Show 2 preset alternatives (e.g. "Minimal: 2 specialists + QA" and
  "Full: all 14 roles") plus let the user type a custom list via "Other".

**Role validation:** After any customisation, validate all role names against the
role catalogue in `${CLAUDE_SKILL_DIR}/references/role-catalogue.md`.
If any name does not match a known role, use AskUserQuestion:
- **Question**: "These roles aren't in the catalogue: {list}. What should we do?"
- **Header**: "Roles"
- **Options**:
  - `Drop them` — Remove unrecognised roles from the team
  - `Keep as custom` — Include them and generate a generic specialist prompt
  - `Re-enter` — Go back to team configuration

**Team completeness check:** After finalising the team, if no Tier 2 (agile) or
no Tier 3 (adversarial) roles remain, use AskUserQuestion:
- **Question**: "Your team has no {missing tier description}. This may reduce quality. Continue?"
- **Header**: "Warning"
- **Options**:
  - `Continue anyway` — Proceed without the missing tier
  - `Add recommended` — Add the default roles for the missing tier back

### Step 3 — Options, model, branch, and coverage

Present four final questions using a single AskUserQuestion call:

**Question A** — feature toggles (multiSelect: true):
- **Question**: "Which options should be enabled?"
- **Header**: "Options"
- **Options**:
  - `Reconnaissance phase (Recommended)` — Phase 0: researcher + qa-engineer review the spec before implementation begins
  - `Adversarial review (Recommended)` — Include adversarial consultants (architecture-critic, security-auditor, etc.) for critique passes
  - `Worktree isolation` — Run each specialist in a git worktree for isolated changes
  - `Skip tests` — Do not require test coverage targets

**Question B** — model choice (single-select):
- **Question**: "Which model for teammates?"
- **Header**: "Model"
- **Options**:
  - `Sonnet (Recommended)` — claude-sonnet for all teammates (fastest, good for most tasks)
  - `Opus` — claude-opus for all teammates (most capable, slower)
  - `Haiku` — claude-haiku for all teammates (fastest, least capable — use for simple specs)

**Question C** — branch strategy (single-select):
- **Question**: "Which branch should teammates work on?"
- **Header**: "Branch"
- **Options**:
  - `Current branch (Recommended)` — Stay on the currently checked-out branch ({current branch name})
  - `New from spec` — Create a new branch derived from the spec filename (e.g. specs/user-auth.md → feature/user-auth)
  - `Custom` — Specify a branch name (via Other)

Before presenting this question, resolve the current branch name (via `git branch
--show-current` or equivalent) so it can be shown in the option description. If
the user selects "New from spec", derive the branch name from the spec filename
and check whether it already exists. If the branch exists, present a follow-up
AskUserQuestion:
- **Question**: "Branch {name} already exists. What should we do?"
- **Header**: "Branch"
- **Options**:
  - `Use it` — Check out the existing branch and work from it
  - `Rename` — Append a suffix (e.g. feature/user-auth-2)
  - `Custom` — Specify a different branch name (via Other)

**Question D** — coverage target (single-select):
- **Question**: "What test coverage target?"
- **Header**: "Coverage"
- **Options**:
  - `80% (Recommended)` — Standard coverage target
  - `60%` — Lighter coverage for prototypes or spikes
  - `95%` — High coverage for critical or production systems
  - `None` — No coverage requirement

If "Skip tests" was selected in Question A, automatically set coverage to "None"
and skip this question.

## Expansion

With all configuration confirmed, read `${CLAUDE_SKILL_DIR}/references/prompt-template.md`
and expand it by substituting all `{PLACEHOLDER}` values:

- `{SPEC_PATH}` — resolved spec file path
- `{FOCUS}` — user-provided focus instructions (omit block entirely if empty)
- `{ROLES_BLOCK}` — generated from confirmed team roles, grouped by tier
- `{ADVERSARIAL_BLOCK}` — included unless adversarial review was disabled
- `{PHASES_BLOCK}` — generated from phase structure (with or without recon)
- `{STACK}` — inferred from spec content, or "infer from project"
- `{STYLE}` — "infer from project"
- `{COVERAGE}` — confirmed coverage target
- `{BRANCH}` — confirmed branch name (current branch, derived from spec, or user-specified)
- `{MODEL}` — confirmed model choice
- `{TEAM_NAME}` — derived from spec filename in kebab-case, with a focus suffix if focus instructions are present (e.g. specs/user-auth.md → "user-auth", or with focus "webhooks" → "user-auth-webhooks")

The block generation rules are defined in `references/prompt-template.md` under
"Block definitions". Read `references/role-catalogue.md` for full role descriptions
when generating {ROLES_BLOCK} content.

The expanded result becomes your operating instructions as team lead.

## Launch

1. **Create the team** — call the TeamCreate tool with:
   - `team_name`: `{TEAM_NAME}`
   - `description`: one-line summary of what the team is building

   TeamCreate is mandatory. Do NOT skip it. Do NOT spawn any agents before
   calling TeamCreate. It creates the team config and task list that all
   coordination depends on.

2. **You are the team lead.** Follow the expanded prompt template for the
   remainder of this session. The TEAM WORKFLOW section tells you how to
   create tasks, spawn teammates into the team, assign work, coordinate,
   and shut down.

Do NOT spawn a separate lead agent — you are the lead.

## Team Model

Three tiers of roles — read `${CLAUDE_SKILL_DIR}/references/role-catalogue.md`
for full definitions.

**Tier 1 — Implementation specialists** (plan-gated): Submit a full plan to the lead
before writing code. Lead verdicts: APPROVE / APPROVE WITH CONDITIONS / REJECT.
Roles: backend-engineer, frontend-engineer, database-architect, devops-engineer,
mobile-engineer, ml-engineer, platform-engineer.

**Tier 2 — Agile specialists** (intent-note only): One-paragraph intent note, begin on
acknowledgement. Run in parallel with implementation. Roles: qa-engineer,
technical-writer, researcher.

**Tier 3 — Adversarial consultants** (critique only): No deliverables. Structured
verdicts: PASS / CONCERNS / BLOCK. Communicate with lead exclusively.
Roles: architecture-critic, security-auditor, ux-skeptic, performance-devil.

## Reference Files

- `${CLAUDE_SKILL_DIR}/references/prompt-template.md` — Lead-agent prompt template with placeholder variables and block definitions
- `${CLAUDE_SKILL_DIR}/references/role-catalogue.md` — Full role definitions for all three tiers
- `${CLAUDE_SKILL_DIR}/references/plan-template.md` — Implementation specialist plan format and lead approval responses
- `${CLAUDE_PLUGIN_ROOT}/agents/` — One agent identity file per role (14 total)
