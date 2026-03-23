---
name: agent-teams
description: >
  This skill should be used when the user asks to "implement the spec",
  "spin up the team", "launch the agents", "start the agent team",
  "run agent teams", "spawn specialist agents", or wants to set up
  multi-agent orchestration in Claude Code, or says "implement this spec",
  "build the feature from spec". Also trigger when the user asks how to structure
  an agent team, which roles to use, or how plan approval should work
  across a team of Claude Code agents.
---

# Agent Teams

Scaffold and launch a coordinated multi-agent implementation team from a spec file.
A lead agent orchestrates specialist teammates through phased implementation with
plan approval gates, parallel builds, and adversarial review.

**Requires:** Claude Code v2.1.32+, `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` environment variable.

## Pre-flight

Before any configuration, perform these checks in order:

1. **Exit plan mode** — if the current session is in plan mode, call ExitPlanMode
   before proceeding. Do not begin configuration until plan mode is fully exited.

2. **Resolve spec** — look for a file path in the user's message (they may say
   "implement specs/foo.md", pass `--spec path`, or just mention a filename).
   If no path is found, use the current session plan as the spec. If neither
   exists, use AskUserQuestion to ask the user to provide a spec file path.
   Stop if no spec can be resolved.

3. **Read the spec** — load the resolved spec file into context. This is needed for
   type classification and team inference in the next step.

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
  - `Data pipeline / ML` — Data processing, ML models, AI pipelines, or infrastructure/DevOps
  - `Mobile / CLI / other` — Mobile apps, CLI tools, internal platforms, or anything else

Use the full team table below to map the selected type to default roles. For
"Mobile / CLI / other", infer the closest match from the spec content (mobile-engineer
for mobile, platform-engineer for CLI/tooling, devops-engineer for infrastructure).

### Step 2 — Configure team roles

Based on the confirmed spec type, pre-select the default roles from this table:

| Spec type | Implement | Agile | Adversarial |
|---|---|---|---|
| Web application | backend-engineer, frontend-engineer, database-architect | qa-engineer, technical-writer | architecture-critic, security-auditor, ux-skeptic |
| API / backend service | backend-engineer, database-architect | qa-engineer, researcher, technical-writer | architecture-critic, security-auditor, performance-devil |
| Data pipeline / ML | ml-engineer, backend-engineer, database-architect | qa-engineer, researcher | architecture-critic, performance-devil, security-auditor |
| Infrastructure / DevOps | devops-engineer, platform-engineer | researcher, technical-writer | architecture-critic, security-auditor, performance-devil |
| Mobile app | mobile-engineer, backend-engineer | qa-engineer, technical-writer | ux-skeptic, security-auditor, performance-devil |
| Internal tooling / CLI | backend-engineer, platform-engineer | qa-engineer, technical-writer | architecture-critic, performance-devil |

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

### Step 3 — Options and model

Present two final questions using AskUserQuestion:

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

After this step, derive the branch name from the spec filename (e.g. `specs/user-auth.md`
→ `feature/user-auth`). Include the branch name in the expansion output summary.

## Expansion

With all configuration confirmed, read `${CLAUDE_PLUGIN_ROOT}/skills/agent-teams/references/prompt-template.md`
and expand the lead-agent prompt by substituting all `{PLACEHOLDER}` values:

- `{SPEC_PATH}` — resolved spec file path
- `{ROLES_BLOCK}` — generated from confirmed team roles, grouped by tier
- `{ADVERSARIAL_BLOCK}` — included unless adversarial review was disabled
- `{PHASES_BLOCK}` — generated from phase structure (with or without recon)
- `{STACK}` — inferred from spec content, or "infer from project"
- `{STYLE}` — "infer from project"
- `{COVERAGE}` — "80%"
- `{BRANCH}` — derived branch name
- `{MODEL}` — confirmed model choice

The block generation rules are defined in `references/prompt-template.md` under
"Block definitions". Read `references/role-catalogue.md` for full role descriptions
when generating {ROLES_BLOCK} content.

Output the fully expanded prompt to begin the lead agent's work.

## Team Model

Three tiers of roles — read `${CLAUDE_PLUGIN_ROOT}/skills/agent-teams/references/role-catalogue.md`
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

- `${CLAUDE_PLUGIN_ROOT}/skills/agent-teams/references/prompt-template.md` — Lead-agent prompt template with placeholder variables and block definitions
- `${CLAUDE_PLUGIN_ROOT}/skills/agent-teams/references/role-catalogue.md` — Full role definitions for all three tiers
- `${CLAUDE_PLUGIN_ROOT}/skills/agent-teams/references/plan-template.md` — Implementation specialist plan format and lead approval responses
- `${CLAUDE_PLUGIN_ROOT}/agents/` — One agent identity file per role (14 total)
