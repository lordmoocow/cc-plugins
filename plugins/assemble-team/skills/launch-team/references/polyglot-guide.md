# Polyglot guide

Polyglot agents combine closely related roles into a single agent to reduce team
size. This file defines which roles can combine (affinity constraints), when to
combine them (decision algorithm), and how to spawn them (spawn rules).

Combinations are **dynamic** — determined by analysing the spec, not by fixed
lookup tables. The affinity groups below define what is **permitted**; the
decision algorithm determines what is **recommended** for a given spec.

---

## Affinity groups

Roles that can be combined into a single polyglot agent. The primary role
determines the agent identity (`subagent_type` and system prompt). The secondary
role's responsibilities are injected into the spawn prompt.

### Tier 1 (implementation specialists)

| Group | Primary | Secondary | When to combine |
|---|---|---|---|
| Backend + DB | backend-engineer | database-architect | Schema work is straightforward or tightly coupled with API design |
| Backend + Mobile | backend-engineer | mobile-engineer | API is purpose-built for the mobile client; little standalone backend logic |
| Infra + Platform | devops-engineer | platform-engineer | Shared infra concerns with developer tooling; no large platform SDK needed |
| ML + Backend | ml-engineer | backend-engineer | ML serving is the core backend; little non-ML business logic |

### Tier 2 (agile specialists)

| Group | Primary | Secondary | When to combine |
|---|---|---|---|
| QA + Docs | qa-engineer | technical-writer | Documentation needs are modest; test suite and docs can be maintained together |
| QA + Research | qa-engineer | researcher | Investigation and validation are tightly coupled (e.g. evaluating libraries while testing) |
| Research + Docs | researcher | technical-writer | Research findings feed directly into documentation; little separate test work |

### Cross-tier combinations

Cross-tier combinations (e.g. Tier 1 + Tier 2, or Tier 1 + Tier 3) are **not
permitted**. Each tier has a distinct protocol — plan-gated, intent-note, or
critique-only — and mixing them creates ambiguity about which protocol the
polyglot agent follows.

### Tier 3 (adversarial consultants)

Tier 3 roles are **not combined with each other by default**. Each adversarial
consultant has a distinct review lens (architecture, security, UX, performance)
and the phase budget usually allows them to remain separate. Only combine Tier 3
roles if the phase would otherwise exceed 5 agents after all other affinity
groups have been exhausted — and flag this to the user as a tradeoff.

---

## Combination rules

1. A role appears in **at most one** polyglot group per team
2. Maximum **2 roles** per polyglot agent (keeps focus manageable)
3. The primary role determines `subagent_type` (loads that agent's identity file)
4. Tier 1 + Tier 1 is valid; Tier 2 + Tier 2 is valid; cross-tier is not
5. Tier 3 roles stay separate unless the phase budget absolutely requires combining
6. If a role has no meaningful work in the spec, **drop it** rather than combining

---

## Decision algorithm

Apply this per phase when proposing the team composition:

```
For each phase:
  1. List the roles active in this phase.
  2. If count <= 5: no combining needed — use individual specialists.
  3. If count > 5: propose combinations from affinity groups, prioritising:
     a. Roles with the most overlap in this spec's requirements.
     b. Roles where one has light duties (e.g. simple schema → combine DB
        into backend).
     c. Tier 2 roles first (agile specialists combine with least friction).
  4. Continue combining until phase agent count is 3–5.
  5. If still > 5 after exhausting affinity groups: drop lowest-value roles
     and flag the tradeoff to the user.
```

Also consider: if a role has very light duties in the spec, suggest combining it
even when the phase is already under 5 agents. Fewer agents with broader scope
is preferable to many agents with thin responsibilities.

---

## Spawn rules for polyglot agents

When spawning a polyglot agent, use these parameters:

```
Agent tool parameters:
  subagent_type : the PRIMARY role name (e.g. "backend-engineer")
  model         : {MODEL}
  name          : compound name using + separator
                  (e.g. "backend-engineer+database-architect")
  team_name     : {TEAM_NAME}
  prompt        : [standard assignment] + [polyglot supplement below]
  isolation     : "worktree" (only if worktree isolation is enabled)
```

### Polyglot prompt supplement

Append this to the standard spawn prompt for the primary role:

**For Tier 1 (implementation) polyglots:**

```
ADDITIONAL ROLE: {secondary-role-name}
Your scope is expanded to also include: {full scope description from
role-catalogue.md for the secondary role}. This overrides any scope
limitations in your base role that would exclude this work.

Submit ONE unified plan covering both domains. Your Scope section must
explicitly list what you own from each role. If the domains conflict,
flag the conflict in your plan for lead resolution.
```

**For Tier 2 (agile) polyglots:**

```
ADDITIONAL ROLE: {secondary-role-name}
Your scope is expanded to also include: {full scope description from
role-catalogue.md for the secondary role}.

Send ONE intent note covering all your responsibilities. Prioritise
the work that unblocks implementation specialists first.
```
