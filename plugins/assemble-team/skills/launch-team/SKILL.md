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

Configure the team through the steps below. Infer sensible defaults from the spec
content, but always let the user confirm or override.

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

### Step 2 — Analyse spec and propose team composition

This step dynamically determines which roles are needed and whether any should be
combined into polyglot agents. Read `${CLAUDE_SKILL_DIR}/references/polyglot-guide.md`
for affinity groups, combination rules, and the decision algorithm.

#### Step 2a — Dynamic spec analysis

After confirming the spec type in Step 1, analyse the spec content in detail:

1. Identify the **work domains** involved (backend, frontend, database, infra, ML, etc.)
2. Determine which roles from the role catalogue are actually needed for this spec —
   drop roles that have no meaningful work (e.g. no frontend-engineer for a pure API)
3. Map the needed roles to phases (recon, foundations, parallel build, integration,
   hardening) based on what depends on what
4. Count agents per phase
5. Apply the **decision algorithm** from `polyglot-guide.md`:
   - If any phase exceeds 5 agents, propose combinations from affinity groups
   - If a role has light duties, suggest combining it even when under 5 agents
   - Prioritise: most overlapping roles first, Tier 2 before Tier 1, never cross-tier
6. Produce a phase-by-phase summary showing: which agents are active in each phase,
   which are polyglot (and what roles they combine), and the agent count per phase

**Target: 3–5 agents per phase.** This is the primary sizing constraint.

#### Step 2b — Present composition to user

Present the proposed team using AskUserQuestion:

- **Question**: "Based on this spec, here's the suggested team:\n\n{phase-by-phase
  summary with agent counts and any polyglot combinations explained, e.g.
  'Phase 1: backend-engineer+database-architect (polyglot), frontend-engineer — 2 agents'}\n\n
  Total unique agents: {N}. Adjust?"
- **Header**: "Team"
- **Options**:
  - `Use suggested (Recommended)` — Accept the dynamic composition as proposed
  - `Adjust combinations` — Modify which roles are combined or split
  - `Full specialist team` — One agent per role, no combining
  - `Custom` — Specify exact composition manually

If the user selects "Adjust combinations", present a follow-up AskUserQuestion:
- **Question**: "Which combinations would you like to change?"
- **Header**: "Adjust"
- **Options** (multiSelect: true): List each proposed polyglot group as "Split
  {roleA} + {roleB} into separate agents", plus any uncombined roles that share an
  affinity group as "Combine {roleA} + {roleB}". Up to 4 options; the user can also
  type custom groupings via "Other".

If the user selects "Full specialist team", revert to the default specialist roles
from the team table in Step 1 (one agent per role, no combining).

If the user selects "Custom", present:
- **Question**: "Describe your desired team composition"
- **Header**: "Custom"
- **Options**: Show 2 preset alternatives (e.g. "Minimal: 2–3 polyglot agents" and
  "Balanced: 4–5 agents with selective combining") plus let the user type a custom
  list via "Other".

#### Step 2c — Role adjustment

After the composition is set (from 2b), allow fine-tuning with AskUserQuestion:

- **Question**: "Final team: {role list, showing polyglot groups as 'A + B'}. Make changes?"
- **Header**: "Roles"
- **Options**:
  - `Looks good (Recommended)` — Proceed with this team
  - `Add roles` — Add additional specialists
  - `Remove roles` — Drop roles or polyglot groups that aren't needed
  - `Custom` — Specify exact changes

If the user selects "Add roles", present a follow-up AskUserQuestion:
- **Question**: "Which roles to add?"
- **Header**: "Add"
- **Options**: List up to 4 roles NOT already in the team (individually or via a
  polyglot group), picking the most relevant based on the spec. The user can also
  type custom role names via "Other".

If the user selects "Remove roles", present:
- **Question**: "Which roles to remove?"
- **Header**: "Remove"
- **Options**: List up to 4 roles or polyglot groups from the current team
  (prioritise the least relevant to the spec). Removing a polyglot group drops
  both constituent roles.

**Role validation:** After any customisation, validate all role names against the
role catalogue in `${CLAUDE_SKILL_DIR}/references/role-catalogue.md`. Polyglot
compound names (e.g. "backend-engineer+database-architect") are valid if both
constituent roles are in the catalogue.
If any name does not match a known role, use AskUserQuestion:
- **Question**: "These roles aren't in the catalogue: {list}. What should we do?"
- **Header**: "Roles"
- **Options**:
  - `Drop them` — Remove unrecognised roles from the team
  - `Keep as custom` — Include them and generate a generic specialist prompt
  - `Re-enter` — Go back to team configuration

**Team completeness check:** After finalising the team, if no Tier 2 (agile) or
no Tier 3 (adversarial) roles remain (counting polyglot groups that cover a tier
as satisfying that tier), use AskUserQuestion:
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
- `${CLAUDE_SKILL_DIR}/references/polyglot-guide.md` — Polyglot affinity groups, decision algorithm, and spawn rules
- `${CLAUDE_SKILL_DIR}/references/skill-map.md` — Maps roles to marketplace skills for enhanced agent capabilities
- `${CLAUDE_PLUGIN_ROOT}/agents/` — One agent identity file per role (14 total)
