# To Discuss — open items not yet folded into DESIGN.md

Parking lot for design questions raised in conversation but not yet decided. Each
item should either become a §5 decision (and move out of here) or be dropped.

---

## 1. Azure-native cost vs. cheaper third-party immutable copy

**Raised:** 2026-06-30.

**Question:** Is the Azure-native 3-2-1(-1-0) configuration expensive, and are there
cheaper third-party targets?

**Where the cost concentrates** — not "3-2-1" generically, but three baseline choices:
- **GRS backup storage** — geo-redundant RSV storage is ~2× LRS, paid on every
  recovery point (the single-vault GRS backplane, §5 2026-06-25).
- **Locked immutability × retention floor** — irreversible; commits to paying for
  every recovery point (incl. LTR) for its full locked life (§5 2026-06-28). Biggest
  driver, compounds with GRS.
- **Protected-instance fees + Cross-Region Restore premium + restore egress.**

The cost is concentrated in the **immutable + off-site copy** — exactly the digits
where third-party is cheapest.

**Two cheaper levers (both already live in the §2 media-diversity ladder):**

- **Lever 1 — cheaper immutable target (S3-compatible Object Lock).** This *is* the
  Maximum (++) tier. Route the deep/long-retention immutable copy off-Azure instead
  of paying GRS+LTR rates:
  | Target | ~Price | Notes |
  |---|---|---|
  | Wasabi | ~$7/TB/mo, no egress/API fees | Object Lock; 90-day min storage duration *aligns with the 90-day rec* |
  | Backblaze B2 | ~$6/TB/mo | Object Lock; cheapest raw; smaller blast-radius separation from Azure |
  | Cloudflare R2 | ~$15/TB, zero egress | Object Lock; good if restores are frequent |

- **Lever 2 — replace the backup engine** (drives the above, often cheaper than
  per-instance Azure Backup at scale): **Veeam** (Backup for Azure / B&R) is the
  common ransomware-resilient fit — writes WORM-locked copies to Wasabi/B2/S3 Object
  Lock or a hardened Linux repo. Also MSP360/CloudBerry (cheapest licensing),
  Commvault Cloud / Druva (SaaS air-gapped, not the cheap option).

**The catch — must preserve the §4 threat model (compromised production Azure admin):**
- **Object Lock in *compliance* mode, not governance mode** — governance is
  admin-reversible and fails the "locked, not unlocked" requirement the same way an
  unlocked RSV policy does.
- **Backup engine control plane + storage credentials outside the production blast
  radius** — a Veeam server or its S3 keys in the same compromised tenant reintroduces
  the closed vector.
- Accept **minimum-storage-duration billing** (Wasabi/B2 bill ~90 days even on early
  deletion) — aligns with the 90-day recommendation anyway.

**Tentative recommendation (not yet a decision):** don't rip out Azure-native. Keep
the locked GRS RSV as the operational/fast-restore copy; make the deep long-retention
immutable copy a **Veeam → Wasabi/B2 Object Lock (compliance mode)** target instead of
paying GRS+LTR for it. Simultaneously buys the **provider independence** that §2 says
only Maximum (++) gives — stronger residual posture *and* lower bill.

**Decision needed:** fold this into §2's ladder as a cost-optimization note, or leave
the ladder posture-only and treat cost as an app-team concern?

---

## 2. GRS redundancy is irreversible once the vault is in use — provisioning-order gate

**Raised:** 2026-06-30.

**Constraint (Azure fact):** RSV storage replication type (LRS/GRS/GZRS) can only be
changed **before the first backup item is configured**. Once anything is protected,
the setting is greyed out.

- It is **not immutability** that freezes redundancy — it's the **presence of
  protected items**.
- **Immutability compounds it:** the only escape from a wrong redundancy choice is to
  *empty the vault* (delete all recovery points) and reset. **Locked immutability
  removes that escape hatch** — recovery points can't be deleted, so the vault can
  never be drained. Net: **locked immutability + protected items on GRS ⇒ the GRS
  choice is permanent.** Two irreversible commitments stacking.

**Forces a strict provisioning sequence (wrong order ⇒ only fix is a brand-new vault):**
1. Create RSV → **set redundancy (GRS/GZRS) first**
2. Enable Cross-Region Restore (needs GRS/GZRS)
3. Enable immutability → **lock it**
4. *Then* start protecting items

**Why this strengthens the third-party case (cross-ref item 1):** with Veeam → Wasabi/
B2 Object Lock, redundancy/geo is a **tunable, not a one-way door** (change bucket
replication, add a region, re-tier without draining), and Object Lock retention is
**independent of the storage redundancy choice** rather than stacked into one
irreversible knob. The RSV irreversibility is itself an argument against putting all
weight on the locked-GRS-RSV backplane.

**Decision needed:** add the provisioning-order gate to §5 (the single-vault decision
currently treats GRS+locked as a posture, not an ordered irreversible gate), and add
the irreversibility trade-off to the §2 ladder.
