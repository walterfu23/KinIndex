# Initial Specs

**Status:** Draft (repo bootstrap)  
**Date:** 2026-07-15  
**Scope:** Founding product definition; **v1 = simplified PoC** of major components  
**Working product name:** KinIndex *(provisional — may change)*  
**Canonical domain term:** **item** (photo or video; used in UI and code)

---

## 1. Product summary

This product is a multi-user orchestration service for family photos and videos. Each user has an account and a private library index of **items** (each item is a photo or a video). The service coordinates making media accessible to Google models and recording the resulting tags so users can understand, browse, and find what they have — **without the service itself viewing or storing the media**.

**North-star outcome:** By analyzing the user’s **historical** family photos and videos, the user ends up with knowledge of **who** appears in their archive, plus **what** they were doing, **when**, and **where**. For videos, that knowledge is often tied to **key points** inside the item. Multi-year coverage comes from analyzing accumulated historical media, not from a timeline-first product framing.

### Terminology

| Term | Meaning |
|------|---------|
| **Item** | A single photo **or** video in the user’s library (end-user and internal/code name) |
| **Photo** / **Video** | The item’s `type` |
| **Library** | The user’s private index of items, annotations, and comments |

### Critical privacy rule

User items are **highly personal** and must be kept private.

| Service does | Service does **not** |
|--------------|----------------------|
| Authenticate users | Look at / inspect user photos or videos |
| Orchestrate ingest, model access, and tagging jobs | Store user item bytes (no media archive on this service) |
| Store account data, item **references**, job status, and **tags** | Proxy, cache, or retain raw media “for convenience” |
| Issue short-lived grants so the client/Google can move or analyze media | Require media to transit this service’s servers |

**Invariant:** Media stays with the user (local / user cloud) and, when needed for analysis, with Google under user-authorized access. This service is an orchestrator and tag index — never a media vault or media viewer backend for the bytes themselves.

---

## 2. Problem

Families already hold large **historical** collections of photos and videos on phones, cameras, and cloud backups. Most of that archive is unorganized: people know *that* they have items, but not *who* is in them. Manual tagging does not scale across years of backlogged media. Users need automatic analysis of that historical library so person tags (and related knowledge) are applied to items they already have.

---

## 3. Goals

1. Support many users, each with their own account and private item library.
2. Accept photos and videos as first-class **items** under a user’s account — especially existing/historical media, not only newly captured items.
3. Automatically analyze media into **who / when / what / where** knowledge (persons first; activities next).
4. Produce durable **person tags** on historical items (and video key points) so users know who appears in their archive.
5. Make retrieval practical: browse and find items by person, activity, place, and date; support manual comments.
6. Keep each user’s items and tags private to that account (no social feed in v1).
7. Uphold the critical privacy rule: the service **orchestrates only** and **never looks at or stores** user item media.
8. Support media that already lives on the user’s device **or** in the user’s own cloud storage.
9. Ensure that, for analysis, a Google model (e.g. Gemini) can **access** the media — tagging cannot run against media the model cannot read.
10. When media must move for analysis, use **client → Google** (or user-cloud → Google) paths only — never via this service.

---

## 4. Non-goals (v1)

- Social network features (public profiles, sharing feeds, public comments)
- Professional media production / video editing suite
- Team or organization workspaces (accounts are individual; family sharing is later)
- Guaranteed perfect recognition of every person/object in every item
- Replacing the user’s entire cloud backup product on day one (focus is tagging + knowledge over raw storage parity)
- This service acting as media storage, CDN, or thumbnail host for user photos/videos
- Service employees, admins, or application servers viewing user item contents

---

## 5. Core concepts

| Concept | Definition |
|--------|------------|
| **Account / User** | An authenticated person who owns a private library |
| **Item** | A photo or video belonging to one user (canonical name in product + code) |
| **Tag** | Structured knowledge about an item — primarily **who / when / what / where** |
| **Person tag (who)** | Basic tag: a person/family member appearing in the item (e.g. “Sam”, “Mom”) |
| **Activity tag (what)** | What persons were doing (e.g. birthday party, swimming, dinner) |
| **When** | Time context — item capture time and/or timestamp within a video |
| **Where** | Place / setting when detectable (e.g. park, home, beach) |
| **Key point** | A moment in a **video** (timestamp or range) to which who/when/what/where (or a comment) can be linked |
| **Comment** | User-written note on an item (or on a video key point); private to the account |
| **Library** | The user’s indexed set of items, tags, key points, and comments (service stores the index, not the media) |
| **Orchestration** | Service coordinating grants, jobs, and tag results without handling media bytes |
| **Auto-tagging** | Google-model analysis of an item; service only triggers the job and stores returned structured tags |
| **Media source** | Where the user’s original file lives: **local** (device/filesystem) or **user cloud** (user’s existing cloud storage) |
| **Model-accessible media** | A copy or reference the chosen Google model can read for tagging (outside this service’s storage) |
| **Direct upload** | Client sends media to Google (or a Google-accessible location) without routing bytes through this service |

### User / account (v1)

- Unique account identity (email or equivalent)
- Authentication (sign up, sign in, sign out)
- Ownership: every item belongs to exactly one user account
- Private by default: User A cannot see User B’s items or tags
- The service never has a reason to open or download user media for human or server-side inspection

### Item fields (v1) — index only

The service persists an **item index record**, not the file:

- `id` — stable unique identifier
- `ownerUserId` — account that owns the item
- `type` — `photo` \| `video`
- `sourceType` — `local` \| `user_cloud` (where the media originated)
- `sourceRef` — optional reference to the user’s original location (local path is client-only; cloud object/URL/id when from user cloud)
- `analysisRef` — reference the Google model uses to access media for tagging (points outside this service; bytes are not stored here)
- `title` — optional user-facing name
- `capturedAt` — when the media was taken (from EXIF/metadata when available, supplied by client/Google — not by this service reading the file)
- `uploadedAt` / `updatedAt`
- `processingStatus` — e.g. `pending` \| `awaiting_model_access` \| `processing` \| `tagged` \| `failed`
- `annotations` — structured who/when/what/where tags (item-level and, for video, key-point-level)
- `comments` — user-authored notes (item-level and optionally key-point-level)
- Optional non-media metadata: duration, dimensions (from client or Google APIs — not from this service decoding the file)

**Note:** Long-term “home” of the file remains with the user (local / user cloud) and/or Google for analysis access. This service stores references + annotations/comments only.

### Knowledge model: who / when / what / where

Item knowledge is organized around four questions:

| Dimension | Meaning | Priority |
|-----------|---------|----------|
| **Who** | Persons / family members in the item | **Basic / primary tags** |
| **What** | Activities those persons were involved in | **Important secondary tags** |
| **When** | When it happened (capture time; and/or time within a video) | Core |
| **Where** | Where it happened (place/setting when known) | Core |

#### Persons (who) — basic tags

- Person tags are the foundation of the product
- End state: historical media analyzed so family members are tagged on the items (and video moments) they appear in
- Users may name/confirm people so the same person tag consolidates across the archive

#### Activities (what)

- Activity tags describe what people were doing (not generic object soup)
- Prefer activities tied to persons when possible (e.g. “Sam — swimming”)

#### When / where

- **When (item):** `capturedAt` and related date context from client/metadata when available
- **When (in-video):** timestamps / ranges on key points
- **Where:** place or setting tags when the model (or user) can supply them

#### Video key points

For **videos**, who/when/what/where will often attach to **multiple key points** inside one item, not only the whole file:

- A key point has a `timestamp` (or start/end range)
- Each key point may carry person tags, activity tags, where, and optional comments
- Item-level tags may still summarize the whole video; key points carry the detailed timeline of knowledge

#### Comments (manual)

- Users may manually add **comments** on an item
- For videos, comments may also attach to a key point
- Comments are private account data stored in the service index (text only — not media)

#### Scope & source

- Annotations are scoped to the owning user’s library
- Primary source is automation after media is model-accessible; users refine persons, activities, and add comments
- Multi-year coverage comes from analyzing historical media, not from a timeline-first product framing

---

## 6. Primary user flows

### 6.1 Create an account

1. User signs up with credentials.
2. Account is created with an empty library.
3. User can sign in on later sessions.

### 6.2 Bring items in for analysis (local or user cloud)

Media may already live in different places. Before tagging, the chosen Google model must be able to access the bytes.

#### A. Local files

1. Authenticated user selects photos/videos from the device/filesystem.
2. Client asks the service for a short-lived **direct-upload authorization** scoped to that user (orchestration only).
3. Client uploads **directly to a Google-accessible location** (client → Google). Bytes never touch this service.
4. Service records item index metadata (`sourceType = local`, `analysisRef`, etc.) and queues a tagging job.
5. Gemini (or another Google model) analyzes via `analysisRef` — this service does not open the file.
6. Tag results return to the service; tags appear in the user’s private index.

#### B. User’s cloud storage

1. Authenticated user connects or selects media from their existing cloud storage (provider TBD).
2. Client / Google-side authorization establishes a path such that a Google model can read the media (grant-in-place or user-cloud → Google). Bytes never touch this service.
3. Service records item index metadata (`sourceType = user_cloud`, `sourceRef`, `analysisRef`) and queues tagging.
4. Model analyzes via `analysisRef`; tags return to the service as in (A).

**Access invariant:** Auto-tagging starts only when the model has a usable `analysisRef`.  
**Privacy invariant:** The service orchestrates and stores the tag index only. It does **not** look at or store user items.

### 6.3 Understand the library (who / when / what / where)

1. User opens their private library index of analyzed historical (and new) items.
2. User browses items and sees who/when/what/where — especially person tags and activities.
3. User filters or searches by person, activity, place, and/or date (and media type).
4. User opens an item: **playback/thumbnails from authorized user/Google-held media**; the service supplies annotations, key points, comments, and status.
5. For a **video**, user can scrub/jump via key points that show who/what/where at moments in the item.

### 6.4 Refine persons & knowledge

1. User confirms or names a detected person (e.g. unlabeled person → “Sam”).
2. That person tag consolidates across other historical items/key points as identity is confirmed.
3. User can correct who/what/where tags; updates reflect in browse/filter.

### 6.5 Add comments

1. User opens an item (or a video key point).
2. User adds a private text comment.
3. Comment is stored in the service index and shown with that item/key point.

---

## 7. Auto-tagging (core capability)

### Intent

After historical (or new) media is made accessible to a Google model, analysis produces structured knowledge — **who, when, what, where** — so the user understands their archive.

- **Who (basic):** persons / family members  
- **What (important):** activities those persons were involved in  
- **When / where:** capture time and place/setting when available  

For **videos**, results should favor **key points** along the timeline, each carrying who/when/what/where as appropriate, rather than only a single bag of item-level tags.

By the end state, analyzing historical media yields person (and activity/place) knowledge across the archive. Multi-year coverage comes from the age and breadth of that archive.

### Model access requirement

Google multimodal models can only tag media they can read. Therefore:

- Local files must be delivered to a Google-accessible location (prefer client → Google direct upload).
- Media in the user’s cloud must be linked, authorized, or staged so the model can access it.
- Item metadata alone is never enough; tagging jobs require a valid `analysisRef`.

### Expected annotation output

- **Who** — persons detected; named/consolidated family-member tags across the archive  
- **What** — activities persons were involved in (birthday, swimming, meal, sports, etc.)  
- **When** — item capture/date context; for video, timestamps on key points  
- **Where** — place/setting when detectable  
- Deprioritize generic object laundry-lists unless they clarify who/what/where

### Candidate models

Google Gemini multimodal models are the primary candidates for auto-tagging photos and especially videos:

- **Gemini 2.5 Flash** — likely default for cost/latency on routine tagging
- **Gemini 2.5 Pro** — higher-quality tagging when Flash is insufficient
- **Gemini 3.1 Pro** — stronger option for harder or higher-stakes items

The tagging pipeline should be model-pluggable so the service can choose Flash vs Pro (or newer Gemini variants) by item type, length, cost budget, or failure retry — without changing the product contract (ingest → tags).

### Processing requirements

- Async: making media model-accessible should complete independently of tagging; tagging starts only after access is ready
- Status visible to the user (`pending` / `awaiting_model_access` → `processing` → `tagged` / `failed`)
- Tagging uses `analysisRef` (Google-accessible URI/object). Service workers must not download or decode media into service-controlled storage
- Videos may be analyzed by Gemini as video input and/or sampled frames **on the Google side**, depending on API limits, cost, and quality tradeoffs
- Video analysis should return **key points** (timestamps/ranges) with who/when/what/where attached where possible
- Photos typically carry item-level who/when/what/where (no key-point timeline)
- Failures should be retryable via orchestration when `analysisRef` still exists; retries may escalate to a stronger Gemini model (e.g. Flash → Pro)
- Logs and support tooling must not include media bytes or media previews

Accuracy targets and exact prompt/schema for annotation output (including key-point structure) are implementation decisions; this spec requires Gemini-class multimodal tagging as the intended automation path.

---

## 8. Feature set

**v1 PoC — must work** (major components only; keep the loop simple)

- [ ] User registration and authentication
- [ ] **2FA**
- [ ] Per-account private **library index** (items + status + tags; no media bytes on this service)
- [ ] Ingest from **local files** via **client → Google direct upload** (user bypasses this service for media bytes)
- [ ] Issue scoped upload/access authorizations; register **item index** metadata only (`sourceType`, `analysisRef`, etc.)
- [ ] Orchestrate tagging jobs once `analysisRef` is ready — without the service reading the media
- [ ] Store returned **who / when / what / where** annotations (persons as basic tags)
- [ ] **Named person consolidation** (“Mom”, “Sam”) across the analyzed archive, including user confirm/rename
- [ ] **Batch ingest** of historical media with progress

**Later (after PoC)**

- [ ] Ingest from **user’s cloud storage** (Drive / Photos / etc.)
- [ ] Client-side thumbnails / playback via authorized access to user/Google-held media
- [ ] Filter / browse by person, activity, place, and date
- [ ] Basic search across annotations, titles, and comments
- [ ] Item detail view (annotations + status from service; media from authorized external source)
- [ ] Manual **comments** on items
- [ ] Video **key points** with who/when/what/where linked to timestamps
- [ ] Comments attachable to video key points
- [ ] Manual correction of who/what/where (beyond person naming)
- [ ] Filter by media type (photo vs video) and date
- [ ] Retry failed tagging jobs
- [ ] Delete item index entry and orchestrate deletion of any Google-side analysis objects the user no longer needs
- [ ] Shared family album / multi-account access
- [ ] Smart albums / collections from who/what/where
- [ ] Explicit year/timeline presentations
- [ ] Mobile capture client
- [ ] Export of annotations / comments / library metadata

---

## 9. UX principles

- Account-first: nothing meaningful happens without sign-in.
- Historical-first: the main job is analyzing media the user already has, not only tagging what they capture today.
- Source-flexible: users can start from local files or media already in their cloud.
- Make-accessible → analyze → understand: tagging waits until the Google model can read the media.
- Who/what first: after analysis, users should see **persons** and the **activities** they were in — not generic object lists.
- Videos explain themselves along the timeline: key points carry who/when/what/where.
- Comments let users add private human context the model cannot know.
- Tags should be readable and useful for family life (real names/roles), not raw model jargon.
- Processing state must be honest (don’t pretend tags exist while access or jobs are still pending).
- Privacy is critical: items are personal; accounts are isolated; no social defaults.
- Trust architecture: the service is visibly an orchestrator — it does not look at or store items.
- Language: say **item** (or “items”) in product copy and code; say **photo** / **video** when the type matters.

---

## 10. Technical direction (initial assumptions)

| Area | Assumption |
|------|------------|
| Product shape | Multi-user orchestration service (not a media host) |
| Accounts | Required; item index records are owned by `userId` |
| Domain object | **Item** (`photo` \| `video`) in API, DB, and UI |
| Media origin | User’s **local files** and/or user’s **own cloud storage** |
| Media storage | **Not this service** — user local, user cloud, and/or Google for model access only |
| Model access | Tagging requires the Google model to have access to the media (`analysisRef`) |
| Local ingest | **Client → Google direct upload**; service never receives or stores media bytes |
| User-cloud ingest | Authorize/link/stage user-cloud → Google without this service touching bytes |
| Service role | **Orchestration only:** auth, grants, job queue, annotation/comment index |
| Annotation model | Who / when / what / where; video key points; user comments |
| Service forbidden | Looking at items; storing items; proxying/caching media |
| Tagging | Orchestrated calls to Google models when model access is ready |
| Tagging models | Google Gemini multimodal models (e.g. 2.5 Flash, 2.5 Pro, 3.1 Pro) |
| Privacy | Critical; personal media; account isolation; zero media retention on this service |
| Clients | Web app first; native later if needed |

Exact auth provider, which user-cloud providers to support first, and app framework are left to implementation planning. Architecture must preserve the orchestration-only boundary.

---

## 11. Success criteria

### v1 (PoC)

1. Users can create accounts, sign in, and use **2FA**.
2. Each user has a separate private **library index**.
3. A user can **batch**-select local photos/videos; the client uploads **directly to Google** for analysis (bytes never transit this service).
4. The service orchestrates tagging and stores returned who/when/what/where tags in the index — it does not look at or store media.
5. Users can **name/confirm persons** so the same person tag consolidates across analyzed items.
6. Job/progress state is visible for batch ingest and tagging (not a silent empty tag set).

*(Browse/filter UI, comments, video key points, user-cloud ingest, and playback/thumbnails are later — not required to call the PoC successful.)*

### End-state outcome (product success)

8. After analyzing historical media, the user has **person tags (who)** on those items, plus meaningful **activity (what)**, **when**, and **where** where available.
9. For videos, that knowledge is linked to **key points** within the item, not only a single item-level summary.
10. The user can select a person and find the items/moments they appear in within the analyzed library.
11. Throughout, personal media remains private: the service only orchestrated; it did not view or store the items.

---

## 12. Open questions

1. Auth method: email/password, magic link, OAuth, or a combination?
2. Which user cloud providers are in MVP (Google Drive / Google Photos / others)?
3. For user-cloud media: grant-in-place vs copy/stage into a Gemini-accessible location?
4. After tagging, must a Google-side copy persist, or can analysis media be ephemeral if the source remains local/user-cloud?
5. Direct-upload mechanism for local files: signed URL, resumable upload, or another Google-supported pattern?
6. How does the service learn that media is model-accessible (client callback vs Google notification/event)?
7. How do person identities form: fully automatic, user-named from suggestions, or clustering + confirm?
8. How rich is activity tagging in MVP vs later (item-level only vs person-linked activities)?
9. Video key-point schema: single timestamps vs ranges; max key points per item; user-editable key points?
10. Default Gemini model for MVP tagging (2.5 Flash vs 2.5 Pro vs 3.1 Pro), and when to escalate?
11. Video tagging strategy with Gemini: native video input vs keyframe/scene sampling for key points?
12. Plan tiers / quotas (jobs, index size) without implying the service stores media?
13. Should users be able to share annotation indexes with family members in v1, or strictly private?
14. Retention of **annotations/comments/index** vs orchestration of deletion for any Google-side analysis copies?
15. How to handle abuse/safety without the service inspecting media (rely on Google-side controls + metadata-only signals)?
16. Where do thumbnails live if the service must not store media (signed Google URLs, client-generated, etc.)?
17. Final product / repo name (KinIndex is provisional)?

---

## 13. Glossary

- **Account / User** — authenticated owner of a private library index  
- **Item** — a photo or video owned by a user (bytes never stored by this service); canonical UI + code term  
- **Who / when / what / where** — core knowledge dimensions for every item  
- **Person tag (who)** — basic tag for a person appearing in an item or key point  
- **Activity tag (what)** — what persons were doing  
- **Key point** — timestamped moment in a video that can carry who/when/what/where and comments  
- **Comment** — user-written private note on an item or key point  
- **Historical media** — photos/videos the user already has from the past / accumulated archive  
- **Library** — a user’s indexed collection of items, annotations, and comments  
- **Orchestration** — service coordinating grants/jobs/results without viewing or storing media  
- **Media source** — local file or user’s cloud storage where the original lives  
- **Model-accessible media / `analysisRef`** — what the Google model can read for tagging  
- **Auto-tagging** — Google-model analysis; service stores the resulting annotations  
- **Direct upload** — client sends media to Google for model access, never through this service  

---

*This document is the initial product specification. Update it as decisions land; prefer amending this file or adding dated follow-on specs under `/Specs` rather than scattering requirements in chat.*
