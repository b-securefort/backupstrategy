# GLOSSARY.md Format — 3-2-1-1-0 Backup Program

The domain glossary lives at **`Documentation/GLOSSARY.md`**.
This file explains how to add or update entries in it.

The program is application-agnostic; the first instance is the **SAP S/4HANA on
Azure IaaS** pilot. Keep definitions reusable across apps, but it's fine to note
a pilot-specific example in the relationship or ambiguity sections.

---

## Purpose

`GLOSSARY.md` is a **glossary only** — not a design spec, not Azure how-to.
It answers: "When someone in a design review or decision says X, what do they
mean in *this backup program*?"

Only **program-specific or overloaded** terms belong. Include a term when:
- It is part of the 3-2-1-1-0 rule itself (Copy, Media, Off-site, Offline/Air-gap, Immutable, Zero-error).
- It is an Azure or SAP term that people use loosely or in conflicting ways here
  (snapshot, replica, vault, soft delete, backint, immutability vs WORM, CMK).
- It carries a ransomware-specific meaning (blast radius, control plane, MUA).

Exclude general IT concepts that mean the same thing everywhere and are never
confused here (TCP, RAID, latency). If a term means the same thing to everyone,
don't add it.

---

## Adding a new term

Add a row to the **Language** table:

```markdown
| **Term** | One tight sentence: what it IS, not what it does. | Aliases / wrong words to avoid |
```

Rules:
- Definition is one sentence. If you need two, the term has two meanings — split it.
- "Aliases to avoid" are the wrong words people reach for. List at least one.
- Be opinionated: if two synonyms exist, pick the canonical one and list the other as an alias.
- Where a term is a control with a ransomware purpose, the definition should make
  the *blast-radius* property explicit (e.g. "immutable" = cannot be altered or
  deleted *even by an administrator* for the retention period).

---

## Adding a relationship

Add a bullet to the **Relationships** section:

```markdown
- One **TermA** → many **TermB**
```

Use cardinality (one-to-one, one-to-many). Only add a relationship if it's
non-obvious or frequently confused — e.g. "One **data source** → three **copies**
→ at least two **media**; exactly one of those copies is **immutable**." Not every
association needs to be listed.

---

## Flagged ambiguities

If a term is genuinely used to mean two different things across the program, add a
row to the **Flagged ambiguities** table:

```markdown
| Ambiguous term | One sentence resolving which meaning wins and when. |
```

Expect these in particular (resolve them early — they cause wrong designs):
- **backup vs snapshot vs replica** — a same-subscription snapshot/replica is *not*
  a 3-2-1 copy if it shares the production blast radius.
- **immutable vs air-gapped vs offline** — immutable (WORM) still lives online and
  is control-plane-reachable; air-gapped/offline is not.
- **off-site vs offline** — off-site = different region (the third "1"); offline =
  out of the attacker's reach (the fourth "1"). A geo-replicated copy is off-site
  but not offline.
- **copy (the "3") vs media (the "2")** — three copies can still violate "2" if all
  three sit on the same storage technology.

---

## When to update GLOSSARY.md

- When a new canonical term is agreed during a design discussion or review.
- When a new control or Azure service is introduced into the design.
- When the same term is used inconsistently across two or more reviews.

Do it in the same change as the design update. `GLOSSARY.md` goes stale fast if
updates are batched.

---

## When NOT to update GLOSSARY.md

- To document how a control is configured (use DESIGN.md or a runbook).
- To document a decision and its trade-off (use DESIGN.md §5).
- To rename a term mid-review without team agreement — resolve in the review first.
