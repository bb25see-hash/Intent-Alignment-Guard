---
name: intent-alignment-guard
description: "MANDATORY SELF-CHECK before any write, external, or high-risk action. Invoke when about to Edit, Write, Bash, push, send, delete, or execute anything that affects state outside the immediate read task."
---

# Intent Alignment Guard

Two distinct functions, checked separately at the action boundary:

- **Authorization** — did the user permit this action? (Tiers 1–4 below.)
- **Quality** — does this output meet the encoded bar? (Constraint Library below.)

Consent and quality are never blurred: an authorized action can still FAIL quality, and a high-quality output can still be unauthorized. Both must pass.

---

## Task-Start — Adaptive

Run at the start of every non-conversational task. Skip for purely conversational turns ("thanks", "ok", "what does X mean").

The guard derives what it can, prints it, and asks only what it genuinely cannot determine. Most tasks fire **zero** questions.

---

### Step 1 — Derive (no tool calls)

From the prompt and immediate conversation, derive:

| Field | How |
|---|---|
| Task statement | one sentence, phrased as an action |
| Scope | concrete paths and systems the work touches |
| Tier | 1–4, per Tier Classification below |
| Projection | token estimate, per the Token Budget table below |
| Floor | least-privilege ops restrictions, per Ops Floor below |
| Failure modes | top 3, per Failure Modes below |

---

### Step 2 — Confidence branch

Self-score confidence on **task statement** and **scope** — the two fields where being wrong causes real damage. High confidence means a single reading is clearly correct and the scope is a named, bounded set of paths.

| Condition | Behavior |
|---|---|
| High confidence **and** Tier 1/2 | Print Context Block, start work. **No menu.** |
| Low confidence, **or** Tier 3/4, **or** conflicting prompt signals | Print Context Block, then **one** menu of at most 3 questions. |

**Conflicting prompt signals** — the prompt both implies and rules out the same capability ("update the tracker" plus "don't touch the vault"). Treat as low confidence regardless of the self-score.

Never skip the Context Block. It is printed on every path, including the no-menu path.

---

### Step 3 — Context Block

Plain chat markdown, printed **before** any menu. Full terminal width, wraps correctly, cannot clip. **All readable detail lives here** — never inside menu options.

```
IAG
Task:        [one sentence, phrased as an action]
Scope:       [concrete paths and systems]
Tier:        [N] - [label]
Confidence:  [high / low - reason if low]
Projection:  ~[N]k tokens, ~[N] writes
Floor:       [computed ops restrictions, comma separated]
Watch for:   1) [failure mode 1]
             2) [failure mode 2]
             3) [failure mode 3]
Constraints: [matching entries, labelled (draft) or (graduated), or "none loaded"]
Last session:[one-line recap, or "none found"]
History:     [up to 2 older matches, or "none matching"]
Mode:        [proceed / proposal first / blocked]
```

**Sourcing for the last three fields:**

1. **Constraints** — scan `constraint-log.md` (draft) and `memory/feedback_*.md` (graduated) for entries whose `[domain]` or `How to apply` matches this task.
2. **Last session** — read `C:\Users\csbuc\Documents\Vault\Mega_Vault\Memory Nexus\memory\conversations\index.md`; under this project's `## <project-name>` group take the last-listed entry (newest — slugs are date-prefixed), open its `<slug>.md`, reduce to one line: what was worked on plus any open thread. Fall back to the current project's local `.claude/memory/conversations.md` (last H2) or `.claude/clear-summaries/` (newest file) only if the vault is unreachable. Otherwise "none found."
3. **History** — grep `C:\Users\csbuc\Documents\Vault\Mega_Vault\Memory Nexus\memory\conversations\*.md` for keyword overlap with this task's domain, **across all projects** (the `project:` frontmatter identifies origin). Surface up to 2 as `[date] [project] one-line why-relevant`, skipping the entry already shown. Cross-project matches are the point — call them out. Skip this step entirely when confidence is high and the task is unambiguous.

> **Path note.** The vault root is `C:\Users\csbuc\Documents\Vault\Mega_Vault\` — always absolute, always including `Documents\`. A bare `Vault\Mega_Vault\...` resolves to `C:\Users\csbuc\Vault\Mega_Vault\`, a different directory that exists but contains no Memory Nexus. Source of truth for vault paths is `C:\Users\csbuc\AppData\Roaming\obsidian\obsidian.json`.

The three `Watch for` items are **self-enforced by default**. No menu is needed to activate them.

---

### Step 4 — Menu (only when Step 2 says so)

At most **three** questions in a single `AskUserQuestion` call:

1. **Task interpretation** — only when genuinely ambiguous. Two or three readings, plus "Skip IAG" as the last option. "Other" is added automatically; do not add it.
2. **Failure modes to block** — `multiSelect`, the three generated modes plus "none - don't self-restrict".
3. **Token ceiling** — only when projection exceeds 100k or the task is Tier 3/4.

**Confidence-forced proposal rule:** if the user selects "Other" on the task question, override the pathway to proposal-first regardless of anything else. Note it under `Mode:` as "(confidence override)".

**If the user selects "Skip IAG":** proceed immediately, no further checks. The hook still logs tier.

---

### Rendering rules — measured, not guessed

`AskUserQuestion` clips silently. Nothing wraps. These caps were measured against the live renderer:

| Field | Cap | Evidence |
|---|---|---|
| `label` | **24 characters** | 30 clipped; 24 did not |
| `description` | **25 characters per line, 2 lines max** | 40 and 60 clipped; 24 and 25 did not |
| `preview` | decorative only | must carry nothing required to choose correctly |

**Do not use in any menu field:** runs of dots, ASCII rulers, box-drawing characters, or leading-space indentation. These do not survive rendering — dot runs are collapsed or dropped entirely.

**The structural rule that makes caps secondary:** anything the reader must understand before choosing goes in the Context Block above the menu. Options carry a label and at most two short bullets. If an option needs explaining, the explanation was in the wrong place.

Count characters before every call. There is no runtime linter.

---

## Token Budget

Replaces action counting. One action can be a three-line edit or a 40k-token sweep, so counts carry no cost information.

### Projection

Estimate at task start from the prompt and a scan of target files:

| Input | Estimate |
|---|---|
| File read | bytes / 3.5 |
| Write or edit | 2x the edited region |
| Subagent dispatch | 15k–40k, by breadth |
| Search sweep | 3k–8k |

Render as one line: `Projection: ~18k tokens, ~6 writes`.

### Proxy ledger — and its limitation

The main session has **no tool that reads its own live token count**; `budget.spent()` exists only inside `Workflow` scripts. Maintain a proxy ledger instead — files read with sizes, edits made, subagents dispatched, searches run — and convert with the table above.

- The overrun figure is an estimate. **Always label it `(est.)`.** Roughly ±30% is enough to catch a task that quietly became several times its projection.
- When the gate is about to fire, optionally invoke `caveman:caveman-stats` for a true session reading before displaying the number. One skill call, only at the moment it matters.

### Overrun gate

Fires when the proxy estimate reaches **1.5x** the projection:

```
OVERRUN - projected ~[N]k, now ~[N]k (est.)
Cause:  [what widened the work]
Spent:  [N] reads, [N] edits, [N] subagents
  (a) continue, re-project to ~[N]k
  (b) stop here
  (c) re-scope and restart
```

Option (a) **re-projects** rather than adding a fixed increment; the next gate fires at 1.5x the new figure. This is a scope-confirmation pause, not a hard block.

### Ceiling question

Only when projection exceeds 100k or the task is Tier 3/4:

```
Projection ~140k. Cap it?
- Follow estimate (~140k)
- 2x estimate (~280k)
- Unlimited
- Custom
```

---

## Ops Floor — computed, never asked

Subtract the capabilities the task needs from the capabilities present in the session. Everything remaining is blocked automatically.

```
Session has:  edit, write, git, vault writes, MCP calls,
              subagents, deletion, billing log
Task needs:   edit, write
Floor:        no git, no vault writes, no external calls,
              no subagents, no deletion, no new files
```

Print on one line in the Context Block. **No consent is requested** — least privilege is the default. A floor rule firing produces the BLOCK dialog in Check 2, whose option (b) removes the restriction and retries.

**Capability inventory to classify against:** source edits, new file creation, file deletion, git operations, vault/memory writes, external API and MCP calls, subagent and workflow dispatch, billing log writes.

| Prompt signal | Result |
|---|---|
| Capability not requested or implied | in the floor (least-privilege default) |
| Capability explicitly requested or implied | not in the floor — blocking it would break the task |
| Capability explicitly ruled out ("don't push", "without touching the vault") | in the floor, marked `[explicit]` with the quoted wording |

This replaces the old fixed System Guardrail menu. It is a computation, not a list, which is why it stops repeating.

| Floor rule | Auto-blocks when... |
|---|---|
| no source edits | Edit/Write targets `.py`, `.ts`, `.js`, `.md`, `.yaml`, `.json` in `scripts/`, `.claude/skills/`, or `src/` |
| no vault / memory writes | Write/Edit targets any path containing `Vault\` or `memory\` |
| no external calls | any MCP tool that isn't read-only |
| no git operations | any Bash/PowerShell call containing `git commit`, `git push`, `git pull`, `git checkout` |
| no subagents | any `Agent()` or `Workflow()` call |
| no billing log | Write/Edit to any path containing `Task Completion Ledger` |
| no deletion | any `rm`, `Remove-Item`, `del`, or `unlink` call |
| no new files | `Write` to a path that does not already exist |

---

## Failure Modes — generated per task

Each is a **pair**: a plain-English risk, plus an observable trip condition. Without the trip condition a failure mode is decoration; the pair makes it enforceable at the action gate.

| Risk | Trips when |
|---|---|
| Rewrites whole SKILL.md instead of surgical edits | `Write` to SKILL.md, or one `Edit` replacing >80 lines |
| Adds a question while claiming to streamline | `AskUserQuestion` with more than 3 questions |
| Invents caps instead of using measured ones | an edit introduces a numeric cap other than 24 or 25 |

*(Example only — generate fresh for every task.)*

### Generation rules

Reject and regenerate a candidate that fails any of these:

1. **Concreteness** — names a path, section, number, or named behavior drawn from **this** prompt. "Might over-engineer" fails. "Might rewrite the caps table instead of editing it" passes.
2. **Portability test** — paste the candidate into an unrelated task. If it still reads sensibly, it is generic. Reject. **This is the primary anti-repetition rule.**
3. **Observability** — a tool call or pattern that trips it can be named.
4. **Not a capability toggle** — anything of the form "no X operations" belongs in the Ops Floor, not here.

### Rolling dedup

Append this task's three failure modes to `.claude/skills/intent-alignment-guard/recent-guards.md`, keeping the last ten tasks. Before finalizing, compare candidates against that file; regenerate any that closely match a recent entry. Repetition becomes detectable rather than structural.

### Escalation

If prompt signals conflict, or the task is a Tier 3/4 candidate, call the `advisor` tool (routed to Claude Fable 5 per project model-tier policy) for a second opinion on which failure modes best protect user intent, before finalizing. This is the Judge from the Architecture section made concrete — separate frontier model, never the actor, optimizing only for user intent.

### Visibility

High confidence and Tier 1/2: the three modes are printed in the Context Block and self-enforced, no menu. Otherwise they appear as the `multiSelect` question in Step 4, plus "none - don't self-restrict".

---

## Action Gate — Tier 2–4 only

Run these checks **in order**. Stop at the first that fires.

---

### Check 1 — Scope Creep Detection

Compare the action's target against the derived scope.

**If the target is outside scope:**
```
SCOPE CREEP DETECTED
• Action:          [exact action]
• Target:          [path or system about to be touched]
• Confirmed scope: [scope from the Context Block]
• Options:
    (a) expand scope to include [target] and continue
    (b) skip this action and continue within confirmed scope
    (c) stop task and re-run IAG with updated scope
```

Do not proceed until the user selects an option. Selecting (a) updates the active scope for the rest of the task.

**Scope matching rules:**
- A target is "in scope" if its path starts with a path listed in the Context Block `Scope:` field
- "Project-wide" scope means any path within the current working directory — but NOT `memory/`, `Vault\`, or external systems unless explicitly named

---

### Check 2 — Guardrail Auto-Block

If an Ops Floor rule or an active failure-mode trip condition covers the about-to-happen action, **BLOCK immediately**:

```
BLOCKED - guardrail fired
• Action:    [exact action]
• Guardrail: "[floor rule or failure mode]" - [floor / failure mode]
• Options:   (a) skip this action and continue  (b) remove restriction and retry  (c) stop task
```

Floor rules and their triggers are tabled under Ops Floor. Failure-mode triggers are the right column of the generated pair.

---

### Check 3 — Token Overrun

When the proxy ledger estimate reaches 1.5x the projection, fire the overrun gate from the Token Budget section. Not a hard block — a mandatory scope-confirmation pause.

---

### Check 4 — Diff Preview (Tier 3 git actions only)

Before any Tier 3 git action (`git push`, `gh pr create`, `gh pr merge`, `git push --force`): run `git diff --stat HEAD` and embed in the gate display:

```
GIT DIFF SUMMARY - [N] files changed
[output of git diff --stat, truncated to 20 lines if longer]

About to: [exact git command]
```

Then proceed to the standard gate. The diff is informational only.

---

### Check 5 — Standard Action Gate

Use `AskUserQuestion` (single question):

**Header:** `"Action Gate"`
**Question:** `"[Tier N] About to [exact action]. Is this within what you authorized?"`
**Options:**
- "Yes - proceed"
- "Yes - show proposal"
- "No - stop"

Always show action bullets. Authorization (first three bullets) and Quality are separate checks — both must pass:

```
• Action:      [exact action - Tier N label]
• Authorized:  [what in this conversation justifies it]
• Skipped:     [lower-risk alternatives considered and rejected]
• Guardrails:  [floor rules + active failure modes covering this action, or "none"]
• Quality:     passes  /  FAILS - constraint violated: "[constraint text]"
• Mode:        [proposal / proceed]
```

**If "No" or Other with blocking intent → BLOCK.** Silence is not consent.

**If `Quality` FAILS → REVISE or BLOCK → run Constraint Capture** before retrying.

---

## Task Completion Block

**Before** the billing log and any closing message, emit this block once. Subagents must not emit it.

```
TASK COMPLETE - Action Audit
-----------------------------------------------
Tier-2+ actions taken:    [N]
  [list each action, one line each, with tier label]

Projection vs actual:     ~[N]k projected, ~[N]k actual (est.)

Scope creep events:       [N, or "none"]
  [if any: what was detected and what the user chose]

Guardrail blocks:         [N, or "none"]
  [if any: which rule fired and the resolution chosen]

Overrun gates:            [N, or "none"]

Constraints captured:     [N, or "none"]
  [if any: domain + title of each new constraint-log entry]

Constraints that fired:   [N, or "none"]
  [if any: which loaded constraint caught something, +1 to its Confirmed count]
-----------------------------------------------
```

---

## Tier Classification

| Tier | Examples |
|------|----------|
| **1 — Read-only** (no gate needed) | Read, Grep, Glob, git status/log/diff, WebSearch, WebFetch |
| **2 — Reversible local write** | Edit, Write, git commit, mkdir, pip install, local config |
| **3 — External / visible to others** | git push, gh pr create/merge, Slack/Gmail/Drive MCP, external APIs |
| **4 — Destructive / irreversible** | rm -rf, force push, DROP TABLE, delete data, change permissions |

**Tier 4 rule:** Requires **explicit written authorization** in this conversation. No implicit consent. BLOCK without it.

---

## Formal Proposal (when "Proposal first" is active)

```
ACTION TYPE:     [Tier 1/2/3/4 + label]
PROPOSED ACTION: [exact action - verbatim]
TASK SCOPE:      [what the user authorized]
EVIDENCE:        [what context in this conversation justifies it]
ALTERNATIVES:    [lower-risk paths considered and why rejected]
GUARDRAILS:      [active Ops Floor rules and failure modes
                  covering this action, or "none"]
```

Wait for explicit confirmation before proceeding.

---

## Outcomes

| Outcome | When | What to do |
|---------|------|------------|
| **ALLOW** | Authorization clear; within scope; no guardrail triggered; projection not overrun; quality passes | Proceed |
| **BLOCK** | No clear authorization; Tier 4 without explicit approval; guardrail triggered; or hard quality violation | Stop; explain why; run Constraint Capture if quality-driven |
| **REVISE** | Partial alignment, or quality FAILS a loaded constraint | Execute a safer/corrected form; run Constraint Capture |
| **ESCALATE** | Requires human decision before any action | Ask; do not infer |

---

## Constraint Capture

Run after any **REVISE** or **BLOCK** driven by quality, or whenever the user rejects an output.

1. **Recognize** — state the specific gap between "looks right" and "is correct."
2. **Articulate** — rewrite the gap as a domain-portable constraint. Strip task-specific nouns; phrase it so it fires on the next analogous task.
3. **Encode** — append to `constraint-log.md` (append-only; never overwrite):

```
### [domain] — [constraint title]
Date:        YYYY-MM-DD
Trigger:     [action or output that caused the rejection]
Constraint:  [the rule — portable, not task-specific]
Why:         [underlying reason — domain logic, past incident, business requirement]
How to apply:[when/where this fires in future tasks]
Confirmed:   0  ← increment on each future task where this constraint catches something
Status:      draft
```

**Auto-graduation:** When `Confirmed` reaches **2**, copy the entry to a `memory/feedback_*.md` file using the existing feedback schema (`rule → **Why:** → **How to apply:**`). Set `Status: graduated` in `constraint-log.md` and add it to the `## Graduated` section with a pointer to the new file.

---

## Constraint Library

- **Load** — at Step 3, scan `constraint-log.md` (draft) **and** `memory/feedback_*.md` (graduated). Surface matches in `Constraints:`, labelled accordingly.
- **Capture** — quality FAIL at the Action Gate triggers Constraint Capture, appending a new draft entry.
- **Graduate** — at `Confirmed: 2`, the draft is copied to durable memory.

File paths:
- `.claude/skills/intent-alignment-guard/constraint-log.md` — session-capture, append-only, draft-grade
- `.claude/skills/intent-alignment-guard/recent-guards.md` — last ten tasks' failure modes, for dedup
- `memory/feedback_*.md` — durable institutional knowledge, load-bearing across all projects

---

## Architecture (multi-agent systems)

- **Actor** optimizes for task completion
- **Judge** (separate frontier model) optimizes only for user intent — never the same model as the actor
- Judge sits at the **action boundary**, not at end of task
- Four required outcomes: ALLOW / BLOCK / REVISE / ESCALATE
- Quality layer (constraint library) sits alongside authorization — both checked at action boundary; neither substitutes for the other
- Scope is derived at Step 1 and printed in the Context Block; scope creep detection enforces it during execution
- Ops Floor and failure modes are scoped to the session — they persist until the task completes or the user removes them

**Known weak point:** confidence self-scoring. A guard confidently wrong about scope skips the menu and starts on the wrong thing. Three limits on the damage: the Context Block prints before work begins and can be interrupted; scope creep detection still fires on every Tier 2+ action; Tier 3/4 always forces the menu regardless of confidence. Residual exposure is a high-confidence Tier 2 misread that stays inside its own wrong scope — recoverable, since Tier 2 is by definition a reversible local write.

**Sub-skill:** `action-permissions` — look up, grant, or revoke permissions across skills, cron jobs, routines, and settings.json. Invoke via `Skill("action-permissions")`.
