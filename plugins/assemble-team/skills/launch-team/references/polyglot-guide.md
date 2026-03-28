# Polyglot guide

Polyglot agents combine related roles into a single agent to reduce team size.
This file defines the principles for combining roles, when to do it, and how
to spawn polyglot agents.

Combinations are **fully dynamic** — you (the lead) analyse the spec and compose
the most appropriate blend for each phase. There are no fixed combo menus or
pair tables. The principles below tell you what is permitted and what makes a
good blend; the decision algorithm tells you when to apply it.

---

## Compatibility principles

Roles within the same tier can be combined freely. The question is not *which
roles are allowed* but *which blend best serves this spec*.

### Tier 1 — Implementation specialists

Roles that share data flows, API contracts, or infrastructure concerns are
natural candidates. The heavier the workload overlap in the spec, the better
the blend.

**Signals that roles should combine:**
- One role has light duties in this spec (e.g. schema is trivial → fold database
  into backend)
- Two or more roles work on tightly coupled output (e.g. backend API exists
  solely to serve the mobile client → combine them)
- Separating roles would create more coordination overhead than value

**Examples** (illustrative, not prescriptive):
- `backend-engineer + database-architect` — when schema work is straightforward
  or tightly coupled with API design
- `backend-engineer + database-architect + mobile-engineer` — for API-centric
  mobile apps where the backend is purpose-built for the client
- `ml-engineer + backend-engineer` — when ML serving is the core backend with
  little non-ML business logic
- `devops-engineer + platform-engineer` — shared infra concerns with developer
  tooling and no large standalone platform SDK

### Tier 2 — Agile specialists

All three agile roles (qa-engineer, technical-writer, researcher) are fully
compatible with each other. Combine any or all based on how much independent
work each has in the spec.

**Examples:**
- `qa-engineer + technical-writer` — when documentation needs are modest
- `qa-engineer + researcher` — when investigation and test validation are
  tightly coupled
- `qa-engineer + technical-writer + researcher` — for focused specs where a
  single agile agent can handle all three responsibilities

### Tier 3 — Adversarial consultants

Adversarial roles can combine when the phase budget requires it or when the
spec's review surface is narrow enough that one agent can cover multiple lenses
effectively. Structure the verdict with sub-sections per lens, each with its
own PASS / CONCERNS / BLOCK rating.

**Examples:**
- `architecture-critic + security-auditor` — for backend-heavy specs where
  architecture and security concerns overlap
- `architecture-critic + security-auditor + performance-devil` — when a single
  comprehensive review pass is more efficient than three separate ones
- `architecture-critic + security-auditor + performance-devil + ux-skeptic` —
  full review panel for tight phase budgets

### Cross-tier combinations

Cross-tier combinations are **not permitted**. Each tier follows a distinct
protocol — plan-gated (Tier 1), intent-note (Tier 2), or critique-only
(Tier 3) — and mixing them creates ambiguity about which protocol the polyglot
agent follows.

---

## Combination principles

1. A role appears in **at most one** polyglot per team
2. The **primary role** is whichever has the heaviest workload in the spec —
   it determines `subagent_type` (loads that agent's identity and system prompt)
3. Stay **within tier** — no mixing plan-gated with intent-note or critique-only
4. **No cap** on roles per polyglot — combine as many as the spec warrants
5. If a role has **no meaningful work** in the spec, drop it entirely rather
   than adding it to a polyglot
6. Prefer **fewer, broader agents** over many thin ones — a polyglot with three
   roles and clear ownership is better than three agents with 20% utilisation

---

## Decision algorithm

Apply this per phase when proposing the team composition:

```
For each phase:
  1. List the roles active in this phase.
  2. For each role, assess its workload in the spec:
     - Heavy: dedicated specialist justified
     - Moderate: could combine if a natural partner exists
     - Light: strong candidate for combining or dropping
  3. Identify roles with overlapping concerns, shared data flows,
     or tight coordination requirements — these are blend candidates.
  4. Compose polyglots from blend candidates within the same tier.
     Choose the primary role (heaviest workload in the blend).
  5. Check agent count for the phase:
     - 3–5 agents: good — proceed
     - > 5 agents: combine more aggressively or drop light-duty roles
     - < 3 agents: fine if the spec is focused; no need to inflate
  6. If still > 5 after combining, drop lowest-value roles and flag
     the tradeoff to the user.
```

Also consider: if a role has light duties, suggest combining it even when the
phase is already under 5 agents. Fewer agents with broader scope is preferable
to many agents with thin responsibilities.

---

## Spawn rules for polyglot agents

When spawning a polyglot agent, use these parameters:

```
Agent tool parameters:
  subagent_type : the PRIMARY role name (e.g. "backend-engineer")
  model         : {MODEL}
  name          : compound name using + separator
                  (e.g. "qa-engineer+technical-writer+researcher")
  team_name     : {TEAM_NAME}
  prompt        : [standard assignment] + [polyglot supplement below]
  isolation     : "worktree" (only if worktree isolation is enabled)
```

### Polyglot prompt supplement

Append one `ADDITIONAL ROLE` block per extra role to the standard spawn prompt.

**For Tier 1 (implementation) polyglots:**

```
ADDITIONAL ROLE: {extra-role-name}
Your scope is expanded to also include: {full scope description from
role-catalogue.md for this role}. This overrides any scope limitations
in your base role that would exclude this work.

[repeat for each additional role]

Submit ONE unified plan covering ALL domains you own. Your Scope section
must explicitly list what you own from each role. If any domains conflict,
flag the conflict in your plan for lead resolution.
```

**For Tier 2 (agile) polyglots:**

```
ADDITIONAL ROLE: {extra-role-name}
Your scope is expanded to also include: {full scope description from
role-catalogue.md for this role}.

[repeat for each additional role]

Send ONE intent note covering all your responsibilities. Prioritise
the work that unblocks implementation specialists first.
```

**For Tier 3 (adversarial) polyglots:**

```
ADDITIONAL REVIEW LENS: {extra-role-name}
Also review from this perspective: {full scope description from
role-catalogue.md for this role}.

[repeat for each additional lens]

Deliver ONE verdict per review pass. Structure it with sub-sections
for each lens. Each sub-section gets its own PASS / CONCERNS / BLOCK
rating. The overall verdict is the most severe of the sub-ratings.
```
