# Codex Dispatch — Cross-Provider Adversarial Review

Codex (GPT-5.x) sits alongside Claude in the review topology as a **cross-provider second opinion**. Its value is non-overlap with Claude's blind spots — not redundancy with the in-session Claude reviewers. The `codex-stop-gate` hook blocks completion when substantive code edits ship without a Codex dispatch or a written bypass reason.

This rule loads alongside `code-review-dispatch.md`, `visual-qa-dispatch.md`, and `advisory-agents-dispatch.md`. CodeRabbit owns generic-review territory at PR time; Codex owns *cross-provider* signal at commit time.

## The Default Is Adversarial

**Dispatch `adversarial-review` by default.** Only downgrade to `review` (gentler) if you can articulate a specific de-escalation reason that survives the Red Flags table below. Only skip entirely with a written bypass reason in `/tmp/cc-gates/$SESSION_ID/skip_codex_gate`.

Lazy = pick adversarial. Articulating a downgrade is friction by design.

## When Adversarial Is Mandatory (No Downgrade Allowed)

Any diff touching:

- **Money flow or payment state transitions** — Stripe intents, refunds, capture/authorize, promo budget reconciliation
- **Auth, session, token, or secret handling** — JWT, OTP, login flows, credential storage
- **Schema migrations or data backfills** — `migrations/**`, `menu_sql/**`, `RunPython`, raw SQL
- **Webhook handlers (any provider)** — signature verification, event-id dedupe, idempotent handlers
- **Cost-bearing external API calls** — LLM completions, Stripe API, push notification batches, SMS sends
- **Idempotency, retry, or distributed-lock logic** — `idempotency_key`, `acks_late`, retry decorators, Redis locks
- **Concurrency primitives** — `select_for_update`, transactions across async boundaries, race-prone update flows
- **4+ files in a single coherent change** — broad scope = broader blast radius
- **Anything that lands in front of a user** — push notification copy, SMS bodies, email content, in-app announcements, banner text, modal copy, error messages, marketing/onboarding strings, announcement sheets. Wrong tone or wrong claim has the same blast radius as a bug.

## Red Flags — Rationalizations to Reject

These are the specific thoughts that mean STOP — you're rationalizing your way out of adversarial review:

| Thought | Reality |
|---|---|
| "This is just a small refactor" | Refactors silently change semantics — that's their failure mode. Adversarial. |
| "Tests cover this" | Tests cover what you thought to test. Adversarial finds what you didn't. |
| "CodeRabbit will catch it at PR" | CR is a backstop, not a substitute. Pre-PR signal is cheaper to act on. |
| "The other reviewers already fired" | Same model family = shared blind spots. Codex's value is non-overlap. Adversarial. |
| "It's user-facing copy, not code" | Copy *is* the user surface — wrong tone or wrong claim has the same blast radius as a bug. Adversarial. |
| "This isn't a payment/auth change" | Adversarial isn't reserved for payments. It's the default. Justify the downgrade, don't assume it. |
| "I already know what Codex would say" | If you knew, you'd have caught it. The point is to be wrong. Adversarial. |
| "Background dispatch will slow me down" | `--background` is non-blocking. You're inventing friction. Adversarial. |
| "It's late / the user is waiting" | The gate exists because deadline pressure is exactly when shipping bypassed review burns the user. Adversarial. |
| "I'll run it after the commit" | The gate fires on Stop. After the commit is too late — the bypass discipline is *before* you're done. Adversarial. |

## Announcement Requirement

Always announce in one line *before* invoking. Three valid forms — anything else is silent skipping:

- **Default (adversarial):** `Dispatching Codex adversarial review on the [surface] diff`
- **Downgrade (review):** `Dispatching Codex review (not adversarial) on the [surface] diff because [specific reason that survives Red Flags]`
- **Skip:** `Skipping Codex on this diff because [reason written to skip_codex_gate]`

If you can't fit the reason in one sentence, the reason isn't real — pick adversarial.

## How to Dispatch

Two invocation paths, both model-callable (the `/codex:*` slash commands have `disable-model-invocation: true` and only the human can type them):

### Background diff review (preferred)

> **Path note:** the `codex/*/scripts/...` segment is an unquoted glob — the shell expands it to the installed plugin version. Do NOT wrap the whole path in double quotes; that prevents glob expansion and makes the command fail silently while the bash tracker still sees a "codex-companion.mjs review" substring and falsely marks dispatch.

```bash
node $HOME/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs adversarial-review --background
```

Or for downgrade:

```bash
node $HOME/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs review --background
```

After dispatch, retrieve with:

```bash
node $HOME/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs status
node $HOME/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs result
```

### Plan-mode review (before acting on a non-trivial plan)

Dispatch the `codex:codex-rescue` subagent in read-only diagnose mode. Useful when the Linear Task Agent is about to execute a plan you authored — Codex critiques the plan text before execution, not the diff after.

```
Agent({
  subagent_type: "codex:codex-rescue",
  description: "Codex plan review",
  prompt: "Read-only diagnosis. Critique this plan: [plan text]. Don't write or modify anything."
})
```

## Cap and Discipline

- **One Codex dispatch per stop-gate trigger.** Slots into the existing 2-reviewer cap as the optional 3rd. Don't run review *and* adversarial-review on the same diff.
- **Background only** unless the diff is truly tiny (≤1 file, ≤20 lines).
- **Present findings, never auto-act.** Mirror the Polish QA / Tone QA pattern — show what Codex flagged, group by severity, let the user decide what to fix. The plugin's `codex-result-handling` skill formalizes this; the hook does not.
- **Don't loop.** If Codex flags X, you fix X, do not re-dispatch Codex on the same diff in the same session. That's the loop-drain failure mode.

## How to Use Results

- **Critical** (Codex's terminology) → fix before commit, no exceptions
- **Important** → fix before commit unless you can name a specific reason not to
- **Minor** → note for user, fix if trivial, defer otherwise
- **Cross-provider non-overlap signal** is highest-value: if Codex flags something none of the Claude reviewers flagged, that's the *whole point* — take it seriously even if it sounds vague to you

If you disagree with a Codex finding, push back with technical reasoning. Don't blindly implement, and don't blindly dismiss.

## Bypass Mechanism

To skip the gate, write a real reason to `/tmp/cc-gates/$SESSION_ID/skip_codex_gate`:

```bash
echo "doc-only change, no executable code" > "/tmp/cc-gates/$SESSION_ID/skip_codex_gate"
```

`$SESSION_ID` is provided to hook scripts via the stdin JSON payload but is NOT in the environment of a Bash tool call. From the model side, look up the active session dir via `SID="$(ls -td /tmp/cc-gates/*/ 2>/dev/null | head -1)"`, then write to `"${SID}skip_codex_gate"`.

Valid bypass reasons (each must name *why* the gate is wrong here, not just "skip"):

- "Doc-only change — no executable code"
- "Whitespace / rename only, semantics unchanged"
- "Generated file (e.g. types.generated.ts), source already reviewed"
- "Already covered by [specific fired reviewer] — non-overlap risk low for this surface"
- "Single-line config tweak in unambiguous direction"
- "Codex API quota exhausted until <HH:MM> — service unavailable" (verified by actual `usage limit` error from `codex-companion.mjs`; pair with a second reason naming review coverage on this diff, e.g. CodeRabbit will re-review on push)

Invalid bypass reasons (these will be rejected at review time):

- "Small change" — small ≠ low-risk
- "I'm in a hurry" — gate exists for this case
- "I already reviewed it myself" — that's not how this works
- "Codex was slow last time" — `--background` is non-blocking

## Gate Scope and Known Limitations

The `codex-stop-gate.sh` hook enforces "Codex diff dispatch happened" only for files tracked via `Edit|Write` PostToolUse hooks. Four known gaps the rule expects you to cover with discipline rather than enforcement:

1. **Bash-mediated mutations are invisible to the gate.** `sed -i`, `tee`, generators, formatters, code patches via shell — none of these append to `edited_files`. If you mutate code via Bash on a high-stakes surface (payments / auth / migrations / user-facing copy / etc.), dispatch Codex manually even when the gate is silent. The Red Flags table still applies.

2. **User-facing copy in `*.md`, `*.json`, `*.yml`, `*.yaml`, `*.toml`, `*.txt` is filtered upstream.** Locale strings, slash-command docs, runbook copy, structured config — the upstream `track-edited-files.sh` excludes these to keep visual-qa noise low. But the rule's mandatory-adversarial list explicitly includes user-facing copy and onboarding text. Same discipline: dispatch manually when these surfaces change.

3. **Edits in a sibling repo (not CC's cwd) return "no diff" from Codex.** `codex-companion.mjs` resolves the diff against the cwd's git repo. If you edited a file in `~/.claude/` or any other repo outside the project root, Codex will report "no concrete changed code path to review" because `git diff` in cwd shows nothing. **Recovery is ordered, not menu:** (a) **default** — `cd` into the sibling repo and dispatch from there; (b) **fallback only** — bypass with a written reason naming the cwd mismatch AND a second sentence explaining why option (a) wasn't viable (e.g. "sibling repo has 4 unrelated dirty files that would flood findings AND I can't pre-stage to scope"). "It's just simpler to bypass" is not a valid (a)→(b) downgrade — that's the lazy-path Red Flag. If the sibling repo IS dirty enough to make (a) noisy, dispatch anyway and **filter the result for findings on your changed paths**; noise on unrelated files is a known trade and still beats empty-diff bypass theater. Don't take an empty-diff Codex result as a clean review — it didn't see the edit.

4. **Dispatch BEFORE committing, not after.** `codex-companion.mjs` resolves the diff against the working tree (`git diff`, not `git diff HEAD~1..HEAD`). If you commit and then dispatch, Codex sees only any *other* uncommitted work in the tree — your actual changes are invisible because they're already in HEAD. Symptoms: Codex returns findings on files you didn't touch this session, or returns "no diff" entirely. Recovery: dispatch before `git commit`, or after committing pass an explicit revision range or scope the review by file path so Codex re-reads the committed change. The companion's job tracker treats the result as "review landed" regardless of which diff was actually evaluated, so silently shipping unreviewed work is the failure mode.

5. **Staged but uncommitted changes are also invisible.** `git diff` shows only working-tree changes; `git diff --cached` shows staged ones. If you `git add` and *then* dispatch (e.g. because the pre-commit gate forced you to retry), Codex reads an empty diff and silently approves. Symptom: instant `approve` verdict with "No material findings" on what should be a substantial diff. Recovery: `git restore --staged <files>` to move the diff back into the working tree, dispatch, then re-stage after the result lands. Don't take a fast `approve` on a non-trivial diff at face value — verify the result mentions specific files you changed.

6. **Hung jobs: bail-fast at ~5min of silent log, don't retry the same diff.** `codex-companion.mjs status` showing `phase: verifying` for many minutes while the log file stops growing (`wc -l` stable across checks) is a deterministic hang, not a transient slowdown — the same input reproduces it. Recovery: `node …/codex-companion.mjs cancel <jobId>`, then write a bypass invoking the "API unavailable" clause AND a second sentence naming the actual review evidence on this diff (regression tests passing, mirrors a documented pattern, low-blast-radius surface — at least one concrete claim, not "I reviewed it myself"). **Two cancelled attempts is the cap *per diff*.** A third try on the same diff wastes the same ~10–12min and produces the same hang. The cap resets when the diff changes substantively — a fresh diff with new logic deserves a fresh attempt, since the hang correlates with specific input shape, not with Codex's general availability.

A future iteration may close these gaps via a SessionStart-baselined augmenter. The naive `git status --porcelain` augmenter doesn't work — it false-positives on pre-existing dirty work that the session didn't touch (validated empirically; see this rule's revision history).

## Boundary

This rule covers **commit-time cross-provider review**. It does not cover:

- Visual / accessibility / tone QA → see `visual-qa-dispatch.md`
- Plan or design challenge before brainstorming → see `advisory-agents-dispatch.md` (devil's advocate)
- In-session project-specific Claude reviewers → see `code-review-dispatch.md` and your project's `code-review-<project>.md` overlay
- Generic code review at PR time → CodeRabbit owns this

Codex sits alongside, not on top of, the existing topology.
