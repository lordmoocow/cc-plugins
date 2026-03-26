# Skill map

Maps team roles to marketplace skills that can enhance their work. The lead
checks which skills are available in the current session and includes relevant
ones in each agent's spawn prompt.

Skills are **optional enhancements** — agents function fully without them. But
when a matching skill is available, using it produces better results than working
from scratch.

---

## How to use this map

Before spawning teammates, check the available skills in your session (they
appear in system-reminder messages). For each agent you are about to spawn,
look up their role below and note any matching skills. Include matching skills
in the agent's spawn prompt using the format in the "Spawn integration" section.

---

## Role-to-skill mapping

### Tier 1 — Implementation specialists

**backend-engineer**
- API development, backend frameworks → e.g. `feature-dev`, `claude-api`
- Code quality review → e.g. `simplify`

**frontend-engineer**
- UI component creation, layouts, styling → e.g. `frontend-design`
- Code quality review → e.g. `simplify`

**database-architect**
- Schema design, migration generation → e.g. `feature-dev`
- Code quality review → e.g. `simplify`

**devops-engineer**
- Infrastructure configuration, CI/CD → e.g. `feature-dev`
- Code quality review → e.g. `simplify`

**mobile-engineer**
- Mobile UI, cross-platform components → e.g. `frontend-design`, `feature-dev`
- Code quality review → e.g. `simplify`

**ml-engineer**
- AI/ML integration, Claude API usage → e.g. `claude-api`, `feature-dev`
- Code quality review → e.g. `simplify`

**platform-engineer**
- SDK design, internal tooling → e.g. `feature-dev`
- Code quality review → e.g. `simplify`

### Tier 2 — Agile specialists

**qa-engineer**
- Test generation, test frameworks → e.g. `feature-dev`

**technical-writer**
- Documentation generation → (any documentation-related skills)

**researcher**
- Web search, library evaluation → (any research or web search skills)

### Tier 3 — Adversarial consultants

**security-auditor**
- Security scanning, vulnerability analysis → (any security audit skills)

**architecture-critic**
- Code quality, structural review → e.g. `simplify`

**performance-devil**
- Performance profiling, benchmarking → (any performance analysis skills)

**ux-skeptic**
- UI/UX review, accessibility → e.g. `frontend-design`

---

## Spawn integration

When including skill recommendations in a spawn prompt, append this section:

```
AVAILABLE SKILLS
The following skills are available in your session and relevant to your role.
Use them when they match work you are about to do — they provide specialised
workflows that produce better results than working from scratch.

  • /{skill-name} — {one-line description of what it does}
  [repeat for each matching available skill]

Invoke a skill by using the Skill tool with the skill name. If a skill does
not appear to be available when you try to use it, proceed without it.
```

Only include skills that are:
1. Actually available in the current session (confirmed from system reminders)
2. Relevant to this agent's role (matched via the mapping above)

Do not include skills that are not present. Do not guess at skill names —
only reference skills you have confirmed are available.
