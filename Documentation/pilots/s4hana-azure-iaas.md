# Pilot — SAP S/4HANA on Azure IaaS

Instantiates the app-agnostic **[3-2-1-1-0 Backup Standard](../DESIGN.md)** for the
first application: **SAP S/4HANA on Azure IaaS** (HR & Salary Payments). This doc
holds only what is *specific* to this application; all rules, controls and
program-wide decisions live in the standard.

---

## Data-source inventory & mapping

Both data classes land in the **same locked, GRS, immutable RSV** (standard §2–§3).

| Data source | Storage | Backup method (standard §2 rule) | Notes |
|---|---|---|---|
| HANA database (data + logs) | Managed Disks / ANF | **Azure Backup for SAP HANA (backint)** — the DB's native app-consistent interface | Point-in-time / log replay |
| OS, app binaries, `/sapmnt`, `/usr/sap/trans`, non-DB disks | Managed Disks / Azure Files / ANF | **Azure VM Backup** (+ Azure Files backup for shares) | Image is sufficient |

> Detailed inventory (HANA tenant DBs, exact disk/volume list, interface
> directories) — **TODO** during pilot build.

---

## Pilot parameters

| Parameter | Value | Source |
|---|---|---|
| Criticality tag | (subscription tag value) — **TODO** | Standard enrollment rule |
| Immutable retention | Default 30-day floor; extend as needed; LTR for statutory payroll retention | Standard §3 |
| Encryption | **PMK** (default) unless payroll/PII compliance mandates CMK — **confirm** | Standard §3 / PMK decision |
| Regulated-data residency | Payroll PII residency / region-pair — **TODO** | Standard §4 |
| RPO / RTO | **TODO** — owner-set; drives §6 test pass criteria | Standard §6 |

---

## Recovery testing — pilot specifics

Instantiates the standard's two-tier model (§6). SAP-specific bindings:

- **Tier 1 (restorability):** HANA backup catalog check + restore of the latest backint
  backup to a scratch HANA target.
- **Tier 2 (full rehearsal):** restore backint backup → isolated clean landing zone →
  log replay to point-in-time → **start SAP** → **HANA consistency check** →
  **payroll business validation (a known pay-run reconciles)** → measure vs. payroll RTO.
- **Major-change trigger** for an out-of-cycle Tier 2: HANA upgrade, vault/key change,
  SAP landscape topology change.

---

## Pilot decision log

### 2026-06-25 — Backint for HANA, VM Backup for non-DB disks

Scope: pilot (S/4HANA on Azure IaaS). Instantiates the standard's
*"App-consistent DB backups; image backup for the rest"* decision.
The HANA database's immutable copy is produced by **Azure Backup for SAP HANA
(backint streaming)**, while **Azure VM Backup** covers OS, app binaries, `/sapmnt`
and other non-DB disks — both targeting the same **locked RSV**. Backint, not
VM-level backup, for the database because a disk image of HANA is crash-consistent
only and offers no point-in-time/log replay, so it would satisfy immutability (the
fourth "1") while failing recoverability (the "0"). **Trade-off**: operating and
monitoring the backint agent plus HANA log backups is more effort than a single
VM-level backup, accepted to guarantee an app-consistent, point-in-time restore.

---

## Open items

- Data-source inventory detail (HANA tenant DBs, disk/volume list).
- RPO/RTO numbers (owner-set).
- Payroll PII residency / region-pair decision.
- PMK-vs-CMK confirmation against payroll compliance.
- Criticality tag value for the pilot subscription.
