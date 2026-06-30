# Decision Log Format — 3-2-1-1-0 Backup Program

Decision log entries live in **`Documentation/DESIGN.md` §5 Decision log**, not as separate files.
This is consistent with the founding 2026-06-24 decision: "Living DESIGN.md is the source of truth."

The program is application-agnostic; the first instance is the **SAP S/4HANA on
Azure IaaS** pilot. A decision may be program-wide (applies to every app) or
pilot-specific — state which in the entry.

---

## When to write a decision log entry

All three must be true. If any one is missing, a design-note or runbook comment is enough.

1. **Hard to reverse** — touches a locked immutability (WORM) policy, retention-lock
   duration, region-pair / data-residency choice, encryption-key ownership (CMK vs
   platform key, and who can delete the key), the identity/subscription/tenant
   boundary isolating the backup data plane, or network isolation (private endpoint /
   public-access disablement). A *locked* immutability policy is effectively permanent.
2. **Surprising without context** — a new engineer would re-debate the decision in
   6 months if there's no record of why it was made.
3. **Real trade-off** — at least one concrete alternative was considered and rejected,
   and there is a cost/rigidity/recovery-time price that was knowingly accepted.

---

## Format

Append to the bottom of §5 in `Documentation/DESIGN.md`:

```markdown
### YYYY-MM-DD — Short imperative title (≤ 8 words)

Scope: program-wide | pilot (S/4HANA on Azure IaaS).
One sentence on what was decided. One or two sentences on the most important why,
including which 3-2-1-1-0 digit it serves. **Trade-off**: what was accepted or
given up. If an earlier decision is superseded, start with:
"**Replaces the YYYY-MM-DD '...' decision.**"
```

Maximum 6 sentences total. If it needs more, the decision has multiple concerns — split it.

---

## What NOT to record as a decision log entry

- Tuning a backup window or schedule with no architectural change.
- Adding a data source that fits the already-decided storage-target pattern.
- Cost-tier moves that are reversible with a one-line config change (e.g. Hot → Cool).
- Term clarifications that belong in `Documentation/GLOSSARY.md` instead.
- Routine recovery-test results — those go in the §6 recovery-test log, not §5.

---

## Examples

Good (all three criteria met):
- *"2026-06-24 — Locked 7-year WORM policy on payroll backup vault"*
  Hard to reverse (a locked immutability policy cannot be shortened or removed),
  surprising without context (why *locked*, not unlocked, given the rigidity?),
  real trade-off (permanent storage cost + zero flexibility vs. a guarantee that a
  compromised admin cannot delete payroll backups — the fourth "1" digit).
- *"2026-06-24 — Backup vault in a separate subscription with MUA"*
  Hard to reverse (identity/blast-radius boundary), surprising (why not same
  subscription with RBAC?), trade-off (operational friction of Multi-User
  Authorization vs. removing production admins from the backup blast radius).

Doesn't qualify:
- *"Moved older backups from Hot to Cool tier"* — reversible config change, no trade-off.
- *"Increased the nightly HANA backup window by one hour"* — schedule tuning, not architecture.
- *"Documented what 'air-gapped' means here"* — belongs in `Documentation/GLOSSARY.md`.
