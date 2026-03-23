# Plan template

Structure for implementation specialist plans. Inject this when spawning
implementation specialists or when a teammate requests the plan format.

---

```markdown
## Implementation plan — {ROLE} — Phase {N}

### Scope

What I will do:
  -

What I will not touch:
  -

### Approach

{Strategy and key decisions. Explain why this approach over the alternatives.
Call out any meaningful tradeoffs.}

### File surface

Create:
  -

Modify:
  -

Delete:
  -

### Dependencies

Blocking (cannot start without):
  -

Non-blocking (will integrate later):
  -

### Risks

| Risk | Mitigation |
|------|------------|
|      |            |

### Status: {READY TO BEGIN / WAITING ON: ...}
```

---

## Lead approval responses

**APPROVE** — Teammate begins implementation. Must check back if scope changes.

**APPROVE WITH CONDITIONS** — Named conditions must be addressed during
implementation. Teammate checks back with lead when complete. Work may begin;
phase cannot close until conditions are met.

**REJECT** — Plan has a significant flaw. Lead states what must change. Teammate
revises and resubmits before writing any code.
