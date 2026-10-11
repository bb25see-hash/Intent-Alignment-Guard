# Intent Alignment Guard

![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)
![Python 3.8+](https://img.shields.io/badge/Python-3.8%2B-green.svg)
![Platform: Claude Code](https://img.shields.io/badge/Platform-Claude%20Code-blueviolet.svg)

> **Did the user actually authorize this?**
>
> IAG answers that question before every write, push, delete, or external action — in under 10ms, with no LLM call.

---

## The problem it solves

Claude Code is powerful enough to push code, send Slack messages, delete files, and call external APIs — all in a single session. Without a guard, *ambiguous intent becomes unintended action*.

IAG inserts a structured consent layer between intent and execution. Routine read-only work passes silently. Consequential actions surface a gate. Destructive actions are blocked outright until you say so explicitly.

---

## How it works

```
  User message
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│  /intent-alignment-guard  (task-start)                  │
│                                                         │
│  Derive: task, scope, tier, projection, floor,          │
│          top-3 failure modes                            │
│                                                         │
│  Self-score confidence, then branch:                    │
│    high + Tier 1/2        ->  0 questions               │
│    low / T3 / T4 / conflict ->  1 menu, max 3 Q         │
│                                                         │
│  "Skip IAG" -> bypass all checks immediately            │
└──────────────────────────┬──────────────────────────────┘
                           │  Context Block always printed
                           ▼
                      Claude works
                           │
              ┌────────────┴────────────┐
              │                         │
        Tier 1–2                    Tier 3–4
        (local)                  (external / destructive)
              │                         │
        hook: silent            hook: STOP directive
        or T2 reminder               │
              │                   ┌───┴────────────────────┐
              │                   │  Action Gate           │
              │                   │  • What exactly        │
              │                   │  • What authorized it  │
              │                   │  • Alternatives tried  │
              │                   └───┬────────────────────┘
              │                       │
              │              ┌────────┴────────┐
              │              │                 │
              │           ALLOW             BLOCK / ESCALATE
              │              │
              └──────────────┘
                    Done
```

---

## Tier classification

The hook (`hooks/pretool_intent_guard.py`) classifies every `Bash`, `PowerShell`, `Edit`, and `Write` call before it executes. No LLM call — pure regex pattern matching in <10ms.

| Tier | Label | Examples | Hook response |
|:----:|-------|----------|---------------|
| **1** | Read-only | `git status`, `grep`, `Read`, `WebSearch` | Silent pass |
| **2** | Reversible local write | `Edit`, `Write`, `git commit`, `mkdir` | Reminder + proceed |
| **3** | External / visible | `git push`, `gh pr`, Slack, external APIs | **STOP** — invoke IAG |
| **4** | Destructive / irreversible | `rm -rf`, force push, `DROP TABLE` | **Blocked** (exit 2) |

**Three escalation paths beyond basic tier:**
- **Sensitive paths** — `.env`, `.ssh/`, `settings.json` → auto-escalate to Tier 3
- **Credential content** — API keys, JWTs, PEM keys in write payloads → auto-escalate to Tier 3
- **Chain analysis** — `cmd1 && cmd2 ; cmd3` → highest tier across all segments wins

---

## Task-start protocol

Most tasks open with **zero questions**. The guard derives what it can, prints it, and asks only what it genuinely cannot determine.

**Step 1 — Derive** (no tool calls): task statement, scope, tier, token projection, ops floor, top-3 failure modes.

**Step 2 — Branch on self-scored confidence:**

| Condition | Behavior |
|---|---|
| High confidence **and** Tier 1/2 | Print Context Block, start work. No menu. |
| Low confidence, **or** Tier 3/4, **or** conflicting prompt signals | Print Context Block, then one menu of at most 3 questions. |

Conflicting signals means the prompt both implies and rules out the same capability — "update the tracker" plus "don't touch the vault."

**Step 3 — Context Block**, printed on every path, before any menu:

```
IAG
Task:        Rework the exclusion menu
Scope:       skills/intent-alignment-guard/SKILL.md
Tier:        2 - reversible local write
Confidence:  high
Projection:  ~18k tokens, ~6 writes
Floor:       no git, no vault writes, no external calls
Watch for:   1) rewriting the file instead of editing it
             2) adding a question while streamlining
             3) inventing caps instead of measuring them
Constraints: none loaded
Mode:        proceed
```

**Step 4 — Menu**, only when Step 2 calls for it: task interpretation (only if genuinely ambiguous) · failure modes to block (`multiSelect`) · token ceiling (only above 100k or Tier 3/4).

### Why the detail is chat text, not menu previews

`AskUserQuestion` clips silently and nothing wraps. Measured against the live renderer: **`label` clips past 24 characters, `description` past 25 per line**. Runs of dots and ASCII rulers do not survive rendering at all, so indented preview templates are unreliable.

Tuning caps alone never held. The structural fix is that anything a reader must understand before choosing lives in the Context Block above the menu, where it wraps properly. Options carry a label and at most two short bullets — nothing load-bearing.

---

## Budget is tokens, not actions

An action count carries no cost information: one action can be a three-line edit or a 40k-token sweep.

The guard projects instead — bytes/3.5 per file read, 2x the edited region per write, 15–40k per subagent, 3–8k per search — and prints `Projection: ~18k tokens, ~6 writes`.

An **overrun gate** fires at 1.5x the projection, re-projecting rather than adding a fixed increment. A **ceiling question** appears only when the projection exceeds 100k or the task is Tier 3/4.

One honest limitation: the main session has no tool that reads its own live token count, so the guard maintains a proxy ledger (files read with sizes, edits, subagents, searches) and labels every figure `(est.)`.

---

## Exclusions are computed, not listed

A fixed exclusion menu offers the same options on every task forever, which is the same as offering none. IAG splits the problem:

**Ops floor** — session capabilities minus task-needed capabilities, applied silently, never asked. Least privilege is the default rather than a checkbox.

```
Session has:  edit, write, git, vault writes, MCP calls,
              subagents, deletion, billing log
Task needs:   edit, write
Floor:        no git, no vault writes, no external calls,
              no subagents, no deletion, no new files
```

**Failure modes** — generated fresh per task as a pair: a plain-English risk and an observable trip condition. Without the trip condition a guardrail is decoration.

| Risk | Trips when |
|---|---|
| Rewrites the whole file instead of surgical edits | `Write` to that file, or one `Edit` replacing >80 lines |
| Adds a question while claiming to streamline | `AskUserQuestion` with more than 3 questions |

A candidate is rejected and regenerated unless it names something concrete from *this* prompt, has an observable trigger, and fails the **portability test** — paste it into an unrelated task, and if it still reads sensibly it is generic. Recent entries are logged to `recent-guards.md` so repetition becomes detectable rather than structural.

---

## Permission planes

`/action-permissions` manages what's authorized across four planes — from most to least privileged:

```
  ┌─────────────────────────────────────────────┐  broadest
  │  ~/.claude/settings.json  allow[]           │  all skills · all projects
  ├─────────────────────────────────────────────┤
  │  .claude/settings.json  allow[]             │  this project only
  ├─────────────────────────────────────────────┤
  │  SKILL.md  ## Authorized Actions            │  that skill only  ← preferred
  ├─────────────────────────────────────────────┤
  │  ~/.claude/cron-permissions.json            │  that cron only   ← preferred
  └─────────────────────────────────────────────┘  narrowest
```

**Always grant at the narrowest scope that works.**  
A skill-declared authorization only fires when that skill is active — it never bleeds into unrelated work.

---

## Outcomes

| Outcome | Condition | Action |
|---------|-----------|--------|
| `ALLOW` | Authorization clear; action within scope | Proceed |
| `BLOCK` | Tier 4 without explicit written approval | Stop; explain why |
| `REVISE` | Partial alignment | Execute a safer form (draft vs. send; read vs. write) |
| `ESCALATE` | Requires human decision | Ask — never infer |

> **Tier 4 rule:** explicit written authorization required in this conversation.  
> No implicit consent. No "they didn't say to stop." **Silence is not consent.**

---

## Repo contents

```
claude-intent-alignment-guard/
├── skills/
│   ├── intent-alignment-guard/
│   │   └── SKILL.md          ← task-start protocol + action gate
│   └── action-permissions/
│       └── SKILL.md          ← permission lookup / grant / revoke
├── hooks/
│   └── pretool_intent_guard.py   ← PreToolUse enforcement hook (<10ms, stdlib only)
├── skill-permissions.template.json   ← per-skill auth registry template
├── settings-hook-snippet.json        ← hook registration snippet
└── LICENSE
```

---

## Installation

**1. Copy the skill files**

```sh
mkdir -p ~/.claude/skills/intent-alignment-guard
mkdir -p ~/.claude/skills/action-permissions
mkdir -p ~/.claude/scripts

cp skills/intent-alignment-guard/SKILL.md ~/.claude/skills/intent-alignment-guard/
cp skills/action-permissions/SKILL.md     ~/.claude/skills/action-permissions/
cp hooks/pretool_intent_guard.py          ~/.claude/scripts/
```

**2. Register the hook in `~/.claude/settings.json`**

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|Bash|PowerShell",
        "hooks": [
          {
            "type": "command",
            "command": "python \"/home/you/.claude/scripts/pretool_intent_guard.py\"",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

**3. (Optional) Seed the permissions registry**

```sh
cp skill-permissions.template.json ~/.claude/skill-permissions.json
```

Or skip this — `/action-permissions` will create and manage it for you on first use.

---

## Usage

| Invoke | When |
|--------|------|
| `/intent-alignment-guard` | Start of any non-trivial task |
| `/action-permissions` | Lookup, grant, or revoke a permission across any plane |

---

## Multi-agent note

When building multi-agent systems:

- **Actor** — optimizes for task completion
- **Judge** — separate frontier model; optimizes *only* for user intent; never the same model as the actor
- Judge sits at the **action boundary**, not end of task
- Scope boundaries must be explicit: what can this agent touch, write, delete?

---

## Requirements

- [Claude Code](https://claude.ai/code)
- Python 3.8+ (hook has no dependencies beyond stdlib)
- `gh` CLI (optional — used by `/action-permissions` for cron plane lookups)
