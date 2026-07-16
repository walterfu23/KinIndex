# KinIndex Rules

**Status:** Canonical  
**Date:** 2026-07-15  
**Authority:** Hard constraints for product, architecture, database, and code. Other docs **link here**; they must not redefine these rules with different wording.

**Companions:** [`Specs_Initial.md`](./Specs_Initial.md) (product), [`Tech_Solutions.md`](./Tech_Solutions.md) (tech choices), [`web_and_client_considerations.md`](./web_and_client_considerations.md) (client strategy)

## Table of contents

- [R1 — Never read or store user media](#r1--never-read-or-store-user-media)
- [R2 — Terminology lock (UI ≡ code)](#r2--terminology-lock-ui--code)
- [R3 — Future-looking design (minimize migrations)](#r3--future-looking-design-minimize-migrations)
- [R4 — Persist useful non-media metadata](#r4--persist-useful-non-media-metadata)
- [R5 — Media movement never via KinIndex](#r5--media-movement-never-via-kinindex)
- [R6 — Human confirm and correct person links](#r6--human-confirm-and-correct-person-links)
- [R7 — KinIndex is not a media cloud](#r7--kinindex-is-not-a-media-cloud)
- [Changelog](#changelog)

---

## R1 — Never read or store user media

KinIndex **never reads, inspects, downloads, decodes, proxies, caches, or stores user media bytes** (photos/videos).

| Allowed on KinIndex service | Forbidden on KinIndex service |
|-----------------------------|-------------------------------|
| Account data, item **references**, job status, **tags**, comments, person links, likeness **embeddings** (vectors) | Item file bytes; media archives; thumbnail/media CDN hosted as KinIndex storage; support tools that show media previews from KinIndex-held bytes |

**Invariant:** Media stays with the user (local / user cloud) and, when needed for analysis, with the **model host** under authorized access. KinIndex is an orchestrator and tag/library service — never a media vault or media-bytes backend.

Clients may display media by loading bytes from local disk or user-cloud/model-host URLs. That does **not** permit uploading those bytes into KinIndex storage.

---

## R2 — Terminology lock (UI ≡ code)

**Whatever term is shown to the end user must be the same term in code** (API fields, DB tables/columns, types, variables, routes, tests, and domain logs).

| Allowed | Not allowed |
|---------|-------------|
| UI **tag** → `tag` / `Tag` / `tagging` | UI “tag” but code `annotation` / `label` / `indexEntry` for the same concept |
| UI **item** → `item` / `Item` | UI “item” but code `asset` / `mediaObject` for the library record |
| UI **key period** → `keyPeriod` / `KeyPeriod` | UI “key period” but code `keyPoint` / `chapter` |

- Pick the user-facing word **first**, then name code after it.  
- Specs, tech docs, and UI copy use the same vocabulary.  
- Exception: pure implementation details never shown to users (e.g. SQL join table names, vendor SDK types).  
- Product **brand** `KinIndex` may differ from domain words.

**Canonical domain terms** (meanings: see specs glossary): **Item**, **Photo** / **Video**, **Library**, **Tag** / **Tagging**, **Person**, **Key period**, **Comment**. Prefer **library** + **tags** in product language over “index” for user-visible features and domain types.

---

## R3 — Future-looking design (minimize migrations)

**System design — including database design — must be future-looking.** Prefer shapes that absorb new product/code versions **without** forcing migrations of existing domain data or breaking code contracts.

| Do | Don’t |
|----|--------|
| Model **end-state** entities early (`Item`, `Tag`, `Person`, `KeyPeriod`, refs, link state, embeddings) even if v1 UI is thinner | Ship a throwaway schema that requires rewrite + data conversion for v2 |
| Prefer **additive** evolution (nullable columns, new tables, new enum values with defaults) | Rename/repurpose columns or change field meaning in place |
| Version extensible payloads (`schemaVersion`) so old rows stay readable | Assume every deploy rewrites all rows |
| Keep providers pluggable as data (`sourceType`, model ids) | Bake one provider into irreversible table/column meaning |
| Stable IDs + link tables | Brittle string encodings that need parse migrations |

**Goal:** New code **reads old data as-is** (compat/defaults). Domain data migrations and breaking API renames are last resort. Ops-only DDL (indexes, perf) is fine; avoidable domain/schema churn is not.

---

## R4 — Persist useful non-media metadata

If data is **not media bytes** and may help retries, orchestration, or later features, **prefer storing it** in the database over discarding it “because v1 doesn’t need it yet.”

Examples: `sourceRef`, **`analysisRef`** (reference string only), job status, tag payloads, likeness embeddings, person link state, embedding model id.

Storing **`analysisRef` is required** when known. Storing a reference is **not** storing media. Clearing or TTL’ing the **remote** analysis object is a separate product decision; do not drop the DB field lightly.

---

## R5 — Media movement never via KinIndex

When media must move for analysis or into the user’s cloud:

- **Client → model host** (e.g. Google) for analysis  
- **Client → user’s cloud** for local→cloud copy  
- **Never** local/cloud → **KinIndex servers** → destination  

KinIndex may issue grants, record progress, and store refs only.

---

## R6 — Human confirm and correct person links

Likeness comparison may **propose** that two person objects are the same. The product **must** support:

1. **Human confirmation** of a person-to-person link  
2. **Human correction** of inaccurate linkage (unlink / split / reassign)  

No irreversible silent merges without a user-visible correction path.

---

## R7 — KinIndex is not a media cloud

KinIndex must not become cloud storage for user items. Accepting and retaining item uploads on KinIndex would violate R1 and turn the product into a media host. Durable homes for files are **user local** and/or **user cloud** (and temporary model-host locations for analysis).

---

## Changelog

| Date | Change |
|------|--------|
| 2026-07-15 | Initial canonical rules extracted from specs / tech / client docs (R1–R7) |
