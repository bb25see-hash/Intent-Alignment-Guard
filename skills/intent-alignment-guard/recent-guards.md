# Recent Guards

Rolling record of generated failure modes, last ten tasks. Newest last.

Purpose: before finalizing a task's failure modes, compare candidates against this file. Regenerate any that closely match a recent entry. This is the mechanical half of the anti-repetition rule — the portability test in SKILL.md is the other half.

Trim to the last ten entries when appending an eleventh.

---

## Entry format

```
### YYYY-MM-DD — [task, one line] ([project])

- [risk, plain English] — trips on [observable condition]
- [risk, plain English] — trips on [observable condition]
- [risk, plain English] — trips on [observable condition]
```

Each failure mode is a **pair**: the risk in plain English, and the tool call or pattern that trips it. A risk without a trip condition is decoration and does not belong here.

Entries must survive the four generation rules in SKILL.md — concreteness, the portability test, observability, and not being a capability toggle. A mode that reads sensibly when pasted into an unrelated task is generic; regenerate it rather than logging it.

---

## Entries

<!-- No entries yet. Append below; keep the ten most recent. -->
