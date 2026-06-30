# 3-2-1-1-0 Backup Standard — Design

Living architecture doc for the organisation's **3-2-1-1-0, ransomware-resilient
backup standard** on Microsoft Azure. **Application-agnostic** — it defines the
standard every critical application's backups must meet.

Each application that adopts the standard gets its own instantiation under
[Documentation/pilots/](pilots/). First instance:
[pilots/s4hana-azure-iaas.md](pilots/s4hana-azure-iaas.md).

The deployed Azure configuration is ground truth. When this doc and the
configuration disagree, the doc is wrong — fix it.

---

## §1 Purpose & scope

Define a reusable standard that lets any critical application's backups satisfy all
five digits of 3-2-1-1-0, and make the ransomware-critical digit (the second "1")
enforceable and auditable across Azure.

| Digit | Meaning | Status in this design |
|---|---|---|
| **3** | Three copies (1 primary + 2 backups) | **Baseline (with caveat)** — 1 primary + RSV (local + GRS (Geo-Redundant Storage) geo-replica) = 3 instances. **Caveat:** the geo-replica *shares the RSV control plane*, so it is a **replica, not an independent 3rd copy** (GLOSSARY); a genuine independent 3rd copy needs **Enhanced (+)**, §2 / §5 |
| **2** | Two different media / fault domains | **Met (baseline, with caveat)** — primary ≠ RSV = 2 distinct Azure services. **Caveat:** both *backup* copies (local RSV + GRS geo-replica) sit on **one service (Azure Backup)**, so the recovery path has no media diversity — a single Azure Backup fault is fatal once the primary is compromised; real backup-side 2-media needs **Enhanced (+)**, §2 / §5 |
| **1** | One copy off-site (different region) | **Designed** — RSV **GRS + Cross-Region Restore**, §5 |
| **1** | One copy offline / immutable (WORM) | **Designed** — locked RSV immutability, §3 |
| **0** | Zero errors — recovery verified | **Designed** — two-tier verification, §6 |

---

## §2 Data-source inventory & storage-target map

Maps each protected data source → which 3-2-1-1-0 copy → which Azure storage. The
concrete inventory for a given application lives in its pilot doc; this section
defines the rules that inventory must follow.

**Routing rule (the spine of the standard):** a data source satisfies the fourth
"1" if it is *either* **natively WORM-capable** *or* has a backup copy **routed into**
a natively immutable target (locked RSV or immutable Blob). "Covered" therefore does
not require the source storage itself to support immutability.

**Storage-type capability matrix:**

| Storage type | Off-site capable | Native WORM? | Fourth "1" via routing | Notes |
|---|---|---|---|---|
| **Managed Disks** | Snapshot→Blob, then GRS | ❌ | ✅ Azure VM Backup → locked RSV | Disk snapshots are not WORM and sit in the blast radius |
| **Azure NetApp Files (ANF)** | ✅ cross-region replication | ❌ | ✅ ANF backup / route copy → RSV/Blob | Snapshots deletable by admin |
| **Azure Files** | ✅ GRS | ❌ | ✅ share backup → RSV | Standard tier has no WORM |
| **Blob (WORM)** | ✅ GRS/GZRS (Geo-Zone-Redundant Storage) | ✅ locked WORM | native | DB backup target, archive |
| **RSV / Azure Backup** | ✅ GRS + Cross-Region Restore | ✅ locked immutability | native | Managed immutable target + MUA + soft delete |

Only **Blob-WORM** and **RSV** can *be* the immutable copy; the rest must *route into* one.

> **ANF caution:** Azure NetApp Files snapshots and cross-region replication are fast
> and convenient but are **not immutable** — an admin can delete them. They may serve
> as a primary or media-diversity copy, never as the fourth "1". The immutable copy of
> ANF-hosted data must still come through a routed RSV/Blob copy (and, for databases on
> ANF, the database's own application-consistent backup).

**Data-source backup-method rule:**

| Data class | Backup method | Why |
|---|---|---|
| Databases (data + logs) | The database's **native application-consistent backup** streamed into the immutable target | App-consistent + point-in-time/log replay |
| OS, app binaries, file data | **Image / file backup** (e.g. Azure VM Backup, Azure Files backup) into the immutable target | An image is sufficient for non-DB data |

An image/disk backup is *not* used as a database's recovery copy: an image of a
running database is crash-consistent only, so it would satisfy the fourth "1"
(immutable) while failing the "0" (clean restore).

**Enrollment scope rule:** the locked-immutability standard (§3) applies to RSVs in
**subscriptions tagged critical**, not to every vault. An **Audit-only** Azure Policy
reports critical RSVs that lack locked immutability; locking itself is a manual,
gated action.

**Media-diversity tiers** — "media" in Azure means *distinct storage service / fault
domain* (there is no customer-controlled physical medium). The baseline is met but
thin, so an opt-in ladder is offered for teams wanting end-to-end protection:

| Tier | Design | Adds | Cost / effort |
|---|---|---|---|
| **Baseline** | Primary storage ≠ locked GRS RSV | "2" by definition | Included |
| **Enhanced (+)** | + 2nd immutable copy in **Blob-WORM, different subscription** | backup-side media + independent copy ("3") + admin blast-radius separation | Extra storage + ops |
| **Maximum (++)** | + **off-Azure** immutable copy (S3 (Amazon Simple Storage Service) Object Lock / on-prem WORM) | provider independence — the only tier outside Azure's control plane | Cross-cloud / on-prem ops |

**Recommendation (both tiers optional; Baseline is the default):**
- **Enhanced (+) — recommended for critical applications.** It is the rung that
  delivers a **genuine independent third copy** (the "3") and a **real second medium**
  (the "2"), in a *different subscription* so production admins are outside its blast
  radius. This is what closes the baseline caveat that the GRS geo-replica is a
  replica, not an independent copy.
- **Maximum (++) — recommended where provider independence matters.** It is the only
  tier that survives a tenant- or Azure-wide control-plane event, because the copy
  lives outside Azure's single control plane entirely.

> Residual at baseline and Enhanced: all copies share the single **Azure provider
> control plane** — a catastrophic tenant/provider-level event is closed only by the
> Maximum (off-Azure) tier, which a cloud-only design cannot otherwise escape.

---

## §3 Controls

The ransomware control set that backs the fourth "1":

- **Immutability:** locked/irreversible policy on in-scope RSVs.
- **Retention:** 30-day minimum floor; owners may extend; **LTR (Long-Term Retention)** permitted for
  compliance depth. Locked immutability covers LTR recovery points too.
- **Multi-User Authorization (MUA)** + resource guard on destructive vault operations.
- **Soft delete** enabled; disabling it gated by MUA.
- **Encryption:** **platform-managed keys (PMK)** by default — there is no
  customer-deletable key for a compromised admin to destroy. **CMK** only when a
  compliance/sovereignty requirement mandates customer-held keys, and then with
  purge protection + key-admin separation.
- **Off-site:** RSV configured **GRS with Cross-Region Restore** — the geo-replica
  in the paired region is the off-site copy and is covered by the same lock.

---

## §4 Threat model & blast radius

Design assumption: the adversary has compromised a **production Azure admin
(control-plane)**, not merely a server. A copy satisfies the fourth "1" only if it
survives that adversary. Locked immutability is irreversible by anyone, including a
Global Admin — which is why "locked," not "unlocked," is the standard.

> Regulated-data residency constraints and the off-site region-pair choice are set
> per application — captured in that application's pilot doc.

---

## §5 Decision log

### 2026-06-25 — Locked immutability standard for critical-app RSVs

Scope: program-wide.
Critical applications' Recovery Services vaults — identified by a subscription
criticality tag, not every vault — get a **locked/irreversible** immutability policy,
satisfying the fourth "1" (offline/immutable) against a control-plane compromise.
Locked rather than unlocked because an admin-disable-able policy lets a compromised
admin delete recovery points and defeat the digit. Immutable retention defaults to a
**30-day minimum floor**; owners may extend it and may enable **Long-Term Retention
(LTR)** for compliance depth — locked immutability covers LTR recovery points too.
**Trade-off**: the floor guarantees the immutable window outlasts short-burn
intrusions but commits owners to irreversibly keep (and pay for) every recovery
point, including long LTR points, for its full duration. Paired with MUA + soft
delete (encryption defaults to PMK — see the PMK decision below).

### 2026-06-25 — Audit-only policy for RSV immutability coverage

Scope: program-wide.
Enrollment is governed by a subscription criticality tag plus an **Audit-only** Azure
Policy that reports critical RSVs lacking locked immutability; the lock itself stays a
**manual, gated action**. Audit — not Deny or auto-remediate — because auto-locking is
irreversible and dangerous at scale, and because tag-tampering is moot once a vault is
locked, leaving **omission** (a critical vault never locked) as the only residual risk,
which detection covers. **Replaces** the earlier hardened tag-DB/reconciler idea, which
was unnecessary: a tag edit cannot reverse an already-locked vault. **Trade-off**:
detection without prevention — relies on a human acting on audit findings to lock each
new critical vault.

### 2026-06-25 — App-consistent DB backups; image backup for the rest

Scope: program-wide.
Each database is protected by its **native application-consistent backup interface**
(streamed into the immutable target), while OS, application binaries and non-database
file data are protected by **image/file-level backup** — both landing in the same
locked RSV. Native DB backup, not a disk/VM image, for databases because an image of a
running database is crash-consistent only and offers no point-in-time/log replay, so it
would satisfy immutability (the fourth "1") while failing recoverability (the "0").
**Trade-off**: operating the DB backup agent and its log backups is more effort than a
single image backup, accepted to guarantee an application-consistent, point-in-time
restore.

### 2026-06-25 — GRS locked RSV as the single immutable backplane

Scope: program-wide.
A single **GRS, locked-immutable RSV** with **Cross-Region Restore** is accepted as
the immutable backup backplane: its geo-replica in the paired region serves as the
off-site copy (third "1"), and the lock protects both replicas against deletion.
Accepted rather than mandating a second independent copy because immutability closes
the ransomware-deletion vector on both replicas and PMK removes the shared-key loss
vector. **Trade-off**: a single vault is one administrative/logical fault domain — a
control-plane or platform-level failure affects both replicas at once — accepted as
the baseline; a second independent immutable copy remains an optional enhanced tier
for the most critical apps. Requires MUA + soft delete.

### 2026-06-25 — Platform-managed keys (PMK) as the default

Scope: program-wide.
Backup encryption defaults to **platform-managed keys (PMK)**, not CMK, because PMK
leaves **no customer-controllable key** for a compromised admin to delete or disable —
satisfying "the operator cannot destroy the key" by construction. CMK is used **only**
when a compliance/sovereignty requirement mandates customer-held keys, and then with
purge protection + key-admin separation. **Trade-off**: PMK gives up customer key
control, rotation control, and crypto-shred capability in exchange for removing a
key-destruction blast-radius element and eliminating misconfiguration risk.

### 2026-06-25 — "2 media" baseline thin, with an opt-in diversity ladder

Scope: program-wide.
"Two media" is defined as **two distinct Azure storage services / fault domains**
(cloud has no physical media to diversify); the **baseline** is met by primary storage
≠ managed vault and is accepted as sufficient for most critical apps. Because all
backups then share one service (Azure Backup/RSV), an **opt-in ladder** is offered:
**Enhanced (+)** adds a second immutable copy in **Blob-WORM in a different
subscription** (backup-side media + independent copy + admin separation), and
**Maximum (++)** adds an **off-Azure** immutable copy (S3 Object Lock / on-prem WORM)
for provider independence. **Trade-off**: the baseline accepts that a logical/service
fault in Azure Backup — or a tenant/provider-level event — is a single point of
failure for all backups, closed only by climbing the ladder at added cost. Off-Azure
is the only tier that escapes the single Azure control plane, which a cloud-only
design cannot.

### 2026-06-28 — Immutable depth: 30-day floor, 90-day recommended, no opt-down

Scope: program-wide. Qualifies the *2026-06-25 locked-immutability* decision.
The locked-immutable retention **floor stays 30 days** (the platform's existing
default), **override-up is optional**, and **opt-down below 30 is not permitted**.
The standard's published **pattern recommends 90 days**. The driver is ransomware
**dwell time**: the clean recovery point an app needs after an attack is an *old,
pre-attack* one, only sought once the intrusion is detected — so the immutable
window must outlive **detection latency**. Security has **no MTTD figure**, so the
floor cannot be pegged to a measured number; 90 days is a **conservative assumption
standing in for the missing MTTD**, to be re-pegged if security produces one. Floor
kept at 30 (not raised to 90) because a **locked** policy is irreversible and a
blanket 90-day lock is a permanent cost commitment for every app, whereas
override-up is reversibility-safe. **Residual (explicit)**: apps left at the 30-day
floor have **no clean recovery point against an attacker who dwells >30 days** — a
known, accepted, *documented* gap, not a silent one. Made visible (not enforced) by
extending the Audit-only policy to report actual depth vs. the 90-day recommendation.
**Trade-off**: keeping 30 as the default means most apps inherit the slow-burn
residual unless they consciously extend; accepted in exchange for not committing the
whole estate to an irreversible, costly 90-day lock on an unverified dwell assumption.

### 2026-06-28 — Shared-responsibility split: platform audits, app teams operate

Scope: program-wide.
The platform team has visibility to **storage media**, not **data sources**, so the
standard is published as a **pattern**: the platform provides building blocks (locked
GRS RSV, immutability, MUA, soft delete, PMK) and **audits** adherence, while **app
teams operate their own backups (backint for DBs, image for the rest), perform all
restores, and attest the "0"** (§7). The one required signal across the boundary is a
per-volume **`backup-class` tag** (`db | file | os`); an unset volume defaults to
**db/unverified**, never file, because defaulting to file would false-green a
database. For the "0": **only the app team writes green** (it alone can validate a
clean recovery), **only elapsed time writes red**, and the platform may only **read
vault restore-job telemetry to corroborate** — it never performs restores or
certifies recovery. **Trade-off**: the platform cannot guarantee any app's "0", only
make its absence visible; accepted because attesting recoverability requires
application visibility the platform structurally lacks. The confirmation/attestation
*mechanism* (evidence-referenced, cadence-pegged email loop) is **advisory guidance
in §7, not mandated**.

---

## §6 Operations & recovery testing

The "0 errors" digit: a backup never restored is a hypothesis, not a backup.
Verification is **two-tier** — a cheap frequent check that backups *are restorable*,
and a periodic full rehearsal that the application *actually recovers*.

**Hard rule:** recovery tests restore into an **isolated, clean landing zone** —
never into production. Restoring ransomware-era recovery points into prod risks
re-detonation and destroys forensic state.

> RPO (Recovery Point Objective) and RTO (Recovery Time Objective) targets are owner-set per application (like retention), and are the
> yardstick both tiers measure against.

### Tier 1 — Restorability check (lightweight, frequent)

- **Verifies:** the latest recovery point restores and is within RPO; immutability and
  retention intact; database-level integrity (DB backup catalog + restore of the latest
  application-consistent backup to a scratch target).
- **Pass:** restore job completes, RPO met, recovery point still immutable, no integrity error.
- **Cadence (default, owner-adjustable):** weekly / every backup cycle; automatable.

### Tier 2 — Full recovery rehearsal (heavyweight, periodic)

- **Verifies:** end-to-end recovery, including the off-site copy:
  1. Restore the application-consistent DB backup into the isolated clean landing zone.
  2. Replay logs to a chosen point-in-time.
  3. Start the application; run the application/DB integrity check.
  4. **Business validation** — a representative business transaction reconciles.
  5. **CRR (Cross-Region Restore) failover drill** — rehearse at least one restore *from the GRS geo-replica*
     in the paired region (proves the off-site copy, not just the local one).
  6. Measure elapsed time against the application's **RTO**.
- **Pass:** all steps succeed and RTO is met.
- **Cadence (default, owner-adjustable):** quarterly, and after any major change
  (DB upgrade, vault/key change, topology change).

### Test-restore log

Record every test here; a backup digit is only "green" if its last test passed.

| Date | Tier | Data source | Point-in-time target | RTO target / actual | Result | Notes / remediation |
|---|---|---|---|---|---|---|

**Failed-test rule:** a failed Tier-1 or Tier-2 test is an incident — it opens
remediation and the affected data source's "0" status reverts to red until a passing
re-test. A green config dashboard with no recent passing test does **not** count as "0".

---

## §7 Shared-responsibility model

The platform team manages Azure subscriptions and infrastructure, **not** the
applications running on them. It therefore has visibility to **storage media**, not
**data sources** (it cannot see that a disk holds a database, nor read application
data). The standard is published as a **pattern** app teams follow — the platform
provides building blocks and audits adherence; it does not operate backups or
perform restores on any app's behalf. 3-2-1-1-0 is split across three parties.

| Responsibility | Platform | App team | Security |
|---|---|---|---|
| Locked GRS RSV, immutability, MUA, soft delete, PMK as building blocks | **R/A** | consumes | — |
| Classify each volume `backup-class = db \| file \| os` (the contract interface) | audits | **R/A** | — |
| Database app-consistent backup (backint) into the locked RSV | — | **R/A** | — |
| Image/file backup of non-DB storage into the locked RSV | — | **R/A** | — |
| Perform restores (Tier-1 + Tier-2) | — | **R/A** | — |
| Attest the "0" (clean recovery) with evidence | reads vault telemetry to corroborate | **R/A** | — |
| Flip "0" to red on staleness / no attestation | **R/A** (automation) | — | — |
| Detection-latency / MTTD assumption; detonation timestamp in an incident | consumes | consumes | **R/A** |
| Audit-only reporting: locked-RSV present + depth-vs-90-day recommendation | **R/A** | — | — |

**The contract interface — `backup-class` tag.** The platform cannot see data
semantics, so the one signal it requires from each app is a per-volume tag:
`db | file | os`. Until set, a volume is treated as **db (unverified for "0")**,
never as file — defaulting to "file" would manufacture a false-green on a database.

**The "0" — who writes green, who writes red.** Only the **app team** can write
green, because only it can start the application and validate a clean recovery; only
**elapsed time** writes red (no fresh attestation → red). Neither party can fake
green: the app team can't suppress staleness, and the platform can't certify a
recovery it cannot see. The platform's sole active role is **reading its own vault's
restore-job telemetry** to corroborate that an attested test actually ran.

**Attestation mechanism (advisory — recommended reference implementation, not
mandated).** An automated confirmation request to the named app + business owner,
**pegged to the test cadence** (§6: Tier-1 weekly, Tier-2 quarterly), not a fixed
30 days. The confirmation must **reference the §6 test-restore log entry / restore
job ID** it attests to — a bare "confirm" click measures email responsiveness, not
recoverability, and rebuilds the false-green one layer up. The platform corroborates
the reference against vault telemetry; **no reply → tag flips non-compliant.**
