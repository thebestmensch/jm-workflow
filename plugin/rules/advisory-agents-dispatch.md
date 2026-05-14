# Advisory Agent Integration

When running brainstorming or writing-plans workflows, auto-dispatch advisory agents at the points below.

**Always announce.** One line: "Dispatching the research agent to investigate X" or "Dispatching the devil's advocate to challenge this design — it crosses multiple components."

## Research Agent (during brainstorming)

When a clarifying question reveals a topic needing deep investigation — existing tools, library comparisons, infrastructure fit — dispatch in background:

```
Agent(subagent_type="research-agent", run_in_background=true)
```

Continue brainstorming while it runs. Incorporate the returned 2-3 options + tradeoffs into approach proposals.

**Preflight before tool/vendor/library evaluations.** Elicit constraints that shape the search space *first* (billing model, integration shape, diversity vs. existing tools, self-host preference, scale) — the research-agent skill owns the full prompt list. Constraints surfaced after a research pass invalidate it.

**Don't dispatch for:** user-preference questions (ask them), topics answerable from codebase exploration, simple factual lookups.

## Devil's Advocate (during brainstorming and planning)

After proposing 2-3 approaches OR drafting a plan, evaluate the complexity gate.

**Fire when ANY:**
- Design involves 2+ components or services
- New external dependencies introduced
- Multiple viable approaches with real tradeoffs
- Plan has 3+ tasks or crosses component boundaries

**Skip when ALL:** single-file/config/copy change, user specified exact approach, purely additive with no tradeoffs, trivial scope.

```
Agent(subagent_type="devils-advocate", model="opus", run_in_background=true)
```

Pass: the full design/plan text + relevant codebase context.

**Results:** `PROCEED` → note briefly and continue. `REVISE` → fold revision points into the proposal before presenting. `RECONSIDER` → present the challenge to the user and discuss before proceeding.
