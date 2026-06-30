# GLOSSARY — 3-2-1-1-0 Backup Program

Domain language for the program. When someone in a design review or decision says
a term below, this is what it means *here*. Program-specific or overloaded terms
only — see [.claude/commands/GLOSSARY-FORMAT.md](../.claude/commands/GLOSSARY-FORMAT.md).

---

## Language

| Term | Definition | Aliases to avoid |
|---|---|---|
| **Copy** | An independently-recoverable instance of the data in its own fault *and administrative* domain. | "replica", "redundancy" |
| **Replica** | A redundant duplicate of a copy that shares the original's management plane (e.g. a GRS (Geo-Redundant Storage) geo-replica) — survives hardware/region loss, not logical/control-plane failure. | "second copy", "independent backup" |
| **Immutable (WORM — Write Once Read Many)** | A copy that cannot be altered or deleted — *even by an administrator* — for its retention period. | "read-only", "air-gapped" |
| **Locked immutability** | An immutability policy committed to an irreversible state, so it cannot be disabled or shortened by anyone, including a Global Admin. | "enabled immutability" |
| **Off-site** | A copy in a different region from the primary (the third "1"); survives a site/region loss. | "offline" |
| **Offline / air-gapped** | A copy not reachable over the production control plane at all — stronger than immutable. | "immutable", "off-site" |
| **Blast radius** | The set of data and backups a single compromised identity or failure can reach and destroy. | "scope", "impact" |
| **Backint** | An SAP-defined backup API by which the database streams its own app-consistent, point-in-time backup directly into a target (here, Azure Backup for SAP HANA → RSV (Recovery Services Vault)). | "DB dump", "disk backup" |
| **Snapshot** | A point-in-time disk/volume image that, in the same subscription, sits inside the production blast radius and is not WORM — therefore not a 3-2-1 copy on its own. | "backup" |
| **MUA** | Multi-User Authorization: a second authorized approver is required for destructive vault operations, so no single admin can weaken or delete protection. | "RBAC (Role-Based Access Control)" |
| **Dwell time** | The period an attacker is present and active in the environment before detection and eviction — the window over which backups may be silently corrupted or encrypted. | "breach duration" |
| **Detection latency (MTTD — Mean Time To Detect)** | The time from initial compromise to the moment defenders notice it; the figure the immutable-retention *depth* must outlive so a pre-attack clean copy still exists. | "response time", "MTTR" |
| **Clean recovery point** | A recovery point captured *before* any attacker corruption and proven structurally consistent and content-valid, so restoring it yields uncorrupted data — the only kind of recovery point that can satisfy the "0". | "latest backup", "good backup" |
| **PMK / CMK** | Platform-Managed Key (Microsoft holds & rotates it; no customer-deletable key) vs Customer-Managed Key (customer-held in Key Vault — controllable, but destroyable). | — |
| **Media (the "2")** | In Azure, two *distinct storage services / fault domains* — not physical media; there is no customer-controlled physical medium to diversify. | "disk vs. tape", "device" |
| **Criticality tag** | A subscription tag marking scope for the locked-immutability standard; drives the Audit-only policy. | — |

---

## Relationships

- One **data source** → three **copies** → on at least two **media**; exactly one copy is **immutable**.
- One **immutable copy** (locked RSV) → one **replica** (GRS geo-copy), which serves as the **off-site** copy — *not* a third independent **copy**.
- One **RSV** → many protected **data sources**.
- One **immutable copy**, kept for its full depth → must still contain at least one **clean recovery point** older than **detection latency**, or the "0" is unachievable after a long-dwell attack.

---

## Flagged ambiguities

| Ambiguous term | Resolution |
|---|---|
| copy vs. replica | A GRS geo-replica is the **off-site copy** for the third "1" but **not** an independent copy for the "3"; when independence is required, only a separately-managed copy qualifies. |
| immutable vs. air-gapped vs. offline | Immutable (WORM) is still online and control-plane-reachable; air-gapped/offline is not. The fourth "1" accepts *either*. |
| off-site vs. offline | Off-site = different region (third "1"); offline = out of the attacker's reach (fourth "1"). **GRS is off-site, not offline.** |
| backup vs. snapshot vs. replica | A same-subscription snapshot or replica is not a 3-2-1 copy if it shares the production blast radius. |
| "2 media" in cloud | Means two distinct Azure storage *services / fault domains*, not physical media; primary≠vault meets it at baseline, but all backups in one service (RSV) is the thin spot the diversity ladder addresses. |
| restore success vs clean recovery | A restore *job* completing ("the bytes came back") is **not** the "0"; only a **clean recovery point** — structurally consistent *and* content-valid (e.g. a business check reconciles) — satisfies it. A successful restore of encrypted data is still a failed recovery. |
