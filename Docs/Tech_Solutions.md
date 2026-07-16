# Tech Solutions

**Status:** Draft  
**Date:** 2026-07-15  
**Companion to:** [`Specs_Initial.md`](./Specs_Initial.md)  
**Hard rules:** [`Rules.md`](./Rules.md) (canonical)  
**Scope:** Concrete technologies that can implement the current specs (v1 PoC first)

## Table of contents

- [How to read this document](#how-to-read-this-document)
- [Hard rules](#hard-rules) → [`Rules.md`](./Rules.md)
- [v1 PoC capability → technology map](#v1-poc-capability--technology-map)
- [1. Accounts & 2FA](#1-accounts--2fa)
- [2. Client application](#2-client-application)
- [3. Orchestration API & service role](#3-orchestration-api--service-role)
- [4. Library database](#4-library-database)
- [5. Direct upload & model-accessible media (`analysisRef`)](#5-direct-upload--model-accessible-media-analysisref)
- [6. Auto-tagging models](#6-auto-tagging-models)
- [7. Job queue & batch ingest progress](#7-job-queue--batch-ingest-progress)
- [8. Named person consolidation (filling Gemini’s cross-item gap)](#8-named-person-consolidation-filling-geminis-cross-item-gap)
  - [The gap](#the-gap)
  - [Approach to fill the gap](#approach-to-fill-the-gap)
  - [How to get embeddings without KinIndex viewing media](#how-to-get-embeddings-without-kinindex-viewing-media)
  - [Product flow](#product-flow)
- [9. Later capabilities (not v1 winners — placeholders)](#9-later-capabilities-not-v1-winners--placeholders)
- [10. Recommended PoC stack (summary)](#10-recommended-poc-stack-summary)
- [11. Change log discipline](#11-change-log-discipline)
  - [Changelog](#changelog)

---

## How to read this document

| Label | Meaning |
|-------|---------|
| **Winner** | Current recommendation for KinIndex given today’s specs |
| **Candidates** | Viable alternatives; keep in mind if specs, constraints, or provider strategy change |

**Important:** These recommendations are **spec-dependent**. When `Specs_Initial.md` or [`Rules.md`](./Rules.md) change, revisit this file and update winners.

Google models are the **current** analysis path; keep the **analysis provider pluggable**.

Obey all rules in [`Rules.md`](./Rules.md) (**R1–R7**), especially no media bytes (**R1**), terminology lock (**R2**), future-looking DB (**R3**), and persist `analysisRef` (**R4**).

---

## Hard rules

Canonical constraints: **[`Rules.md`](./Rules.md)**. This document does not redefine them.

---

## v1 PoC capability → technology map

| Spec capability (v1 PoC) | Winner | Role |
|--------------------------|--------|------|
| Account create / sign-in + **2FA** | **Clerk** | Username/password **or** social (e.g. Google); built-in 2FA |
| Private per-user **library** | **PostgreSQL** + **Prisma** + **pgvector** | Items, refs, tags, persons, embeddings — never media bytes; future-looking / additive schema |
| API / orchestration service | **Next.js** (App Router) + TypeScript | Web UI + API routes/server actions in one PoC codebase |
| Client → Google direct upload | **Gemini Files API** (resumable / browser upload) | Media bytes go client → Google; KinIndex only issues grants + records `analysisRef` |
| Tagging orchestration + store tags | **Gemini API** (`gemini-2.5-flash` default) | Multimodal who/when/what/where; results written into Postgres |
| Batch ingest + progress | **Inngest** (or DB job rows in PoC) | Queue/track batch jobs; surface progress to the client |
| Named person consolidation | **pgvector** + **Postgres `Person`** + confirm/correct UI | Likeness suggests; human confirm + unlink/split required |

---

## 1. Accounts & 2FA

**Spec need:** Sign up / sign in with **username/password** **or** external accounts (at least **Google**; other social providers as configured). **2FA** supported. Every library row owned by `userId`.

**Note:** Google as a **sign-in provider** is separate from Google as an **analysis/model** provider. A user may use password-only auth and still upload to Gemini for tagging (via KinIndex/project credentials), or sign in with Google without that implying Drive/Photos ingest.

| | Technology |
|---|------------|
| **Winner** | **Clerk** — username/password + social connectors (Google first); built-in 2FA; session/JWT for API auth |
| **Candidates** | **Auth0** / **Okta CIC** — username/password + many social IdPs; MFA; heavier for PoC |
| | **Supabase Auth** — email/password + Google (and other) OAuth; MFA available |
| | **Firebase Authentication** — email/password + Google and other federated IdPs; solid MFA |
| | **Better Auth** or **Lucia** + OAuth + custom TOTP — full control; more build/maintain cost |
| | **NextAuth.js / Auth.js** — credentials + Google/other providers; 2FA mostly DIY |

**Revisit when:** Specs require additional IdPs (Apple, Microsoft, etc.), passwordless-only, enterprise SSO, or “no third-party IdP.”

---

## 2. Client application

**Spec need:** Web first; batch pick local files; show ingest/tagging progress; person confirm/rename. Browse/filter and playback are **later**.

| | Technology |
|---|------------|
| **Winner** | **Next.js** (React, App Router, TypeScript) |
| **Candidates** | **Vite + React** SPA + separate API |
| | **Remix** / **SvelteKit** / **Nuxt** — fine if team preference shifts |
| | **Expo / React Native** — only when specs prioritize mobile capture client |

**Revisit when:** Specs mandate native-first or offline-heavy local library browsing.

---

## 3. Orchestration API & service role

**Spec need:** Authz, short-lived upload grants, job orchestration, annotation index. **Must not** receive, proxy, or store media bytes.

| | Technology |
|---|------------|
| **Winner** | **Next.js Route Handlers** (same app as UI) for PoC speed |
| **Candidates** | **Hono** or **Fastify** (Node/TS) as a standalone API |
| | **FastAPI** (Python) — strong if Gemini/Vertex SDKs and ML glue dominate |
| | **Go (chi/echo)** — when specs push hard on concurrency/cost at scale |

**Hosting (PoC):** **Vercel** (fits Next.js) or **Fly.io** / **Railway**.  
**Candidates later:** **Google Cloud Run**, **AWS ECS/Lambda** — especially if analysis stays on GCP/Vertex.

**Revisit when:** Specs split “BFF vs worker” or forbid serverless for long tagging jobs.

---

## 4. Library database

**Spec need:** Multi-user private library — items, `analysisRef`, processing status, tags, person entities, embeddings. No media blobs.

**Hard rules:** [`Rules.md`](./Rules.md) **R1** (no media bytes), **R3** (future-looking / additive schema), **R4** (persist useful refs/metadata).

| Practice | Example |
|----------|---------|
| End-state entities from day one | `Item`, `Tag`, `Person`, `KeyPeriod`, `Comment`, link state, embedding columns — even if some UI ships later |
| Stable IDs + link tables | Appearances link `itemId`/`keyPeriodId` ↔ `personId` |
| Extensible documents | Tag payload / provider metadata with `schemaVersion` |
| Provider pluggability | `sourceType`, `analysisProvider`, `embeddingModelId` as data |
| Persist useful refs early | `sourceRef`, `analysisRef`, embedding vectors |
| Additive DDL only (default) | New nullable columns / new tables; avoid renames |
| Compat reads | New code must read old rows without a mandatory rewrite job |

| | Technology |
|---|------------|
| **Winner** | **PostgreSQL** (+ **pgvector** for likeness) |
| **ORM / access** | **Prisma** (winner for TS PoC) — still follow additive migration discipline when DDL is unavoidable |
| **Candidates (ORM)** | **Drizzle**, **Kysely**, raw SQL |
| **Candidates (DB)** | **SQLite** (local demos only — weak multi-user story) |
| | **MongoDB** — flexible JSON tags; weaker relational person linking |
| | **DynamoDB** / **Firestore** — if specs go serverless-only on AWS/GCP |
| **Hosted Postgres candidates** | **Neon**, **Supabase**, **RDS**, **Cloud SQL** |

**Revisit when:** Specs add heavy full-text search across huge libraries (may add **OpenSearch**, **Typesense**, or rely more on Postgres FTS) — prefer **adding** engines beside the core schema, not replacing it.
---

## 5. Direct upload & model-accessible media (`analysisRef`)

**Spec need:** Client bypasses KinIndex for bytes; model must read media; service stores references only.

**Decision:** Persist **`analysisRef` in the database** (and `sourceRef` when known) per [`Rules.md`](./Rules.md) **R4**. Refs are library metadata — not user media (**R1**).

| | Technology |
|---|------------|
| **Winner** | **Google Gemini Files API** — client uploads file; KinIndex **stores** returned file URI as `analysisRef` on the item row |
| **Candidates** | **GCS signed URL (PUT)** → object URI → Gemini/Vertex reads from GCS |
| | **Vertex AI** multimodal input via GCS URI (more “production GCP”) |
| | **Resumable upload** protocols as required by the chosen Google surface |

**Non-goals for v1 (per specs):** user Drive/Photos ingest as the only path — local and mixed sources still apply per hybrid doc.

**Revisit when:** Specs change analysis provider, or require mandatory deletion of the **remote** analysis object after success (DB may still keep a historical/null `analysisRef` + tombstone).

---

## 6. Auto-tagging models

**Spec need:** Multimodal who / when / what / where; photos + videos; pluggable over time.

| | Technology |
|---|------------|
| **Winner** | **Gemini 2.5 Flash** — default cost/latency for PoC batch tagging |
| **Candidates (same family)** | **Gemini 2.5 Pro**, **Gemini 3.1 Pro** — escalate on hard items / retries |
| **Candidates (later / if specs drop Google-only)** | **GPT-4o / GPT-4.1** class vision, **Claude** multimodal, **Vertex open models**, self-hosted VLMs |
| **API surface** | **Google AI Gemini API** (PoC winner) vs **Vertex AI** (candidate for GCP-centric prod) |

KinIndex should call models behind an internal **TaggingProvider** interface (`analyze(analysisRef) → annotations`) so swapping Flash→Pro or Google→other is a provider change, not a product rewrite.

**Revisit when:** Specs require video **key periods**, stricter accuracy, or non-Google providers as first-class.

---

## 7. Job queue & batch ingest progress

**Spec need:** Batch select many historical items; visible progress; tagging async after `analysisRef` is ready.

| | Technology |
|---|------------|
| **Winner** | **Inngest** — durable functions, retries, progress events; fits Next.js PoC |
| **Candidates** | **Trigger.dev** — similar DX |
| | **BullMQ + Redis** — classic, self-hosted control |
| | **Google Cloud Tasks** / **Cloud Workflows** — if stack standardizes on GCP |
| | **Postgres-backed job table** + worker poll — simplest PoC; weaker at scale |
| | **Temporal** — when specs need complex long-running workflows |

**Revisit when:** Specs demand exactly-once semantics, huge fan-out, or strict per-tenant rate limits.

---

## 8. Named person consolidation (filling Gemini’s cross-item gap)

### The gap

**Google Gemini (and similar multimodal LLMs) analyze one item (or one request) at a time.** They can detect people *inside* media1 and *inside* media2 and return boxes / descriptions / tags for each call. They do **not** maintain a durable, library-wide identity that says “person box A in media1 = person box B in media2.”

So KinIndex must **own cross-item person linking**. Gemini remains the **per-item** tagger (who/when/what/where for that file). Likeness + human confirm/correct is a **separate KinIndex layer**.

```
media1 ──► Gemini ──► detections + tags (item-local)
media2 ──► Gemini ──► detections + tags (item-local)
                │
                ▼
     embedding step (not KinIndex viewing pixels)
                │
                ▼
     KinIndex: store vectors → similarity match → suggest links
                │
                ▼
     Human: confirm / correct / name  →  stable personId across library
```

### Approach to fill the gap

| Stage | Who | What |
|-------|-----|------|
| 1. Per-item understanding | **Gemini** | Boxes, who/when/what/where for **this** item only |
| 2. Likeness vector | **Face/person embedding model** (separate from “link my whole library”) | One embedding per detection |
| 3. Cross-item link proposals | **KinIndex** (pgvector similarity within `userId`) | “These two detections look like the same person” |
| 4. Ground truth | **Human** | Confirm or correct inaccurate person-object linkage; name (“Sam”) |

**Hard rules:** Likeness must not cause KinIndex to read media bytes (**R1**). Embeddings are produced on client or analysis path; KinIndex stores vectors only. Human confirm/correct is mandatory (**R6**).

### How to get embeddings without KinIndex viewing media

Pick one (or combine) of these **candidates**:

| Option | How it works | Fit |
|--------|--------------|-----|
| **A. Client-side embed (winner for privacy clarity)** | After Gemini returns boxes, **KinIndex client** loads the item from local/user-cloud, crops faces, calls an embedding API (or on-device model), POSTs **vectors only** to KinIndex | Strong: bytes never touch KinIndex; works with mixed sources the client can already view |
| **B. Analysis-path sidecar on model host** | Same pipeline that can read `analysisRef` runs a **face-embedding** step (Vertex / dedicated model) and returns embeddings with the tag payload | Strong for batch; KinIndex still only receives vectors |
| **C. Ask Gemini for match in a small batch** | Send several images in one prompt: “which faces are the same?” | **Not** the library solution — doesn’t scale; use only as a rare assist |
| **D. Rely on Google Photos “People” API** | If source is Google Photos and it already clustered people | Provider-specific bonus; not general for Dropbox/pCloud/local |

**Recommended default:** **A or B** for embeddings + **KinIndex pgvector** for linking + **required human confirm/correct**. Do not expect Gemini alone to close the gap.

### Product flow

1. Gemini tags each item (item-local person detections + boxes + who/what/where). For videos, tags attach to **key periods** (time ranges); person evidence for embedding is taken **within** the period, not necessarily at `start`.  
2. Embedding step produces a likeness vector per detection (client or analysis sidecar) — photo still, or representative crop(s) inside a video key period.  
3. KinIndex stores embeddings (**not** images) and runs similarity → **suggested** links.  
4. **Human confirmation (required):** accept “same person.”  
5. **Human correction (required):** unlink/split/reassign wrong links.  
6. User names the `personId` (“Sam”); search/overlays use that across media1, media2, …

| | Technology |
|---|------------|
| **Per-item tags** | **Gemini** (Flash/Pro) — does *not* own cross-item identity |
| **Cross-item linker** | **KinIndex + PostgreSQL + pgvector** — similarity within `userId`; link state `suggested` \| `confirmed` |
| **Embedding source (candidates)** | Client-side face embed API / on-device model; **Vertex AI** / dedicated face embedding beside Gemini; avoid “Gemini library graph” as the design |
| **Vector DB candidates** | pgvector (winner) → Pinecone / Qdrant / Weaviate if needed |
| **UI** | Confirm / correct person-object links using viewer overlays |
| **Avoid** | Assuming Gemini links people across the archive; irreversible silent merges; KinIndex downloading pixels to embed |

**Privacy check:** Gap-fill uses vectors + human review. Matching embeddings ≠ KinIndex reading photos.

**Revisit when:** A provider offers a first-class cross-item people graph you choose to import (e.g. Photos people) as an *additional* signal, still with human confirm/correct.

---

## 9. Later capabilities (not v1 winners — placeholders)

Track here so future spec promotions have a starting shortlist:

| Later spec area | Likely candidates |
|-----------------|-------------------|
| User-cloud ingest | Google Drive API, Google Photos Library API, (later) other clouds |
| Browse / filter / search | Postgres indexes → Typesense / OpenSearch / `pg_trgm` |
| Thumbnails / playback | Signed Google/GCS URLs, client File System Access API — **never** KinIndex media hosting |
| Video key periods | Gemini video ranges (`start`/`end`) + richer annotation schema; embeddings from person **within** period |
| Comments | Postgres text on item / key period rows |
| Mobile client | Expo / React Native, or Flutter |

---

## 10. Recommended PoC stack (summary)

```
Clerk (username/password + Google social + 2FA)
    → Next.js / TypeScript (UI + orchestration API)
        → PostgreSQL + Prisma + pgvector (library index, tags, person embeddings, jobs)
        → Inngest (batch + tagging workflows)
        → Gemini Files API (client direct upload → analysisRef)
        → Gemini 2.5 Flash (tagging; Pro as fallback candidate)
```

**Privacy check:** Media bytes flow **browser → Google** only. KinIndex stores accounts, refs, statuses, and tags.

---

## 11. Change log discipline

When updating specs:

1. Diff `Specs_Initial.md` for new/removed capabilities or constraint changes.
2. Mark affected rows in this file **Stale**.
3. Re-pick **Winner** vs **Candidates** (especially analysis provider and upload path).
4. Record the date and a one-line reason in a short changelog below.

### Changelog

| Date | Change |
|------|--------|
| 2026-07-15 | Initial draft aligned to v1 PoC in `Docs/Specs_Initial.md` |
| 2026-07-15 | Auth: username/password **or** social (Google+) sign-up/sign-in |
| 2026-07-15 | Person linking: **likeness comparison** via embeddings + pgvector (primary) |
| 2026-07-15 | Person linking: **human confirm + correct** inaccurate person-object links required |
| 2026-07-15 | Document Gemini **cross-item gap**; KinIndex owns likeness layer (embed + match) |
| 2026-07-15 | **Persist `analysisRef` in DB**; prefer keeping useful non-media index metadata |
| 2026-07-15 | Specs **terminology lock**: UI terms ≡ code terms |
| 2026-07-15 | **Future-looking design**: avoid avoidable code/data migrations |
| 2026-07-15 | Point hard constraints at canonical [`Rules.md`](./Rules.md) |
