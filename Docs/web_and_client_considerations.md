# Web and Client Considerations

**Status:** Draft  
**Date:** 2026-07-15  
**Companion to:** [`Specs_Initial.md`](./Specs_Initial.md), [`Tech_Solutions.md`](./Tech_Solutions.md), [`Rules.md`](./Rules.md)  
**Scope:** Web-heavy vs web-light vs **recommended hybrid** for KinIndex **with the full end-state product in mind**. Phasing may affect *when* a surface ships; it should not redefine *what* the product must eventually support.

**Hard rules:** [`Rules.md`](./Rules.md) (**R1–R7**) — especially never read/store media (**R1**), terminology lock (**R2**), future-looking design (**R3**), media never via KinIndex (**R5**).

## Table of contents

- [End-state requirements (design to these)](#end-state-requirements-design-to-these)
- [Shared constraints](#shared-constraints)
- [Solution A — Web-heavy](#solution-a--web-heavy)
  - [Idea](#idea)
  - [Strengths / weaknesses (short)](#strengths--weaknesses-short)
  - [End-state fitness](#end-state-fitness)
- [Solution B — Web-light](#solution-b--web-light)
  - [Idea](#idea-1)
  - [Strengths / weaknesses (short)](#strengths--weaknesses-short-1)
  - [End-state fitness](#end-state-fitness-1)
- [Comparison (A vs B)](#comparison-a-vs-b)
- [Recommended solution — Hybrid (mixed sources)](#recommended-solution--hybrid-mixed-sources)
  - [Recommendation](#recommendation)
  - [Why hybrid wins for mixed sources](#why-hybrid-wins-for-mixed-sources)
  - [Media source model (index)](#media-source-model-index)
  - [Local → user-cloud copy (feature)](#local--user-cloud-copy-feature)
  - [Analysis path (unchanged boundary)](#analysis-path-unchanged-boundary)
  - [Viewer path (anchor scenario)](#viewer-path-anchor-scenario)
  - [Architecture sketch](#architecture-sketch)
  - [Capability matrix (recommended hybrid)](#capability-matrix-recommended-hybrid)
  - [Product guidance for users](#product-guidance-for-users)
- [Sequencing (toward the hybrid)](#sequencing-toward-the-hybrid)
- [Open decisions (within the hybrid)](#open-decisions-within-the-hybrid)
- [Changelog](#changelog)

---

## End-state requirements (design to these)

| End requirement | What the client(s) must enable |
|-----------------|--------------------------------|
| **Never read/store media** | Service holds only refs, tags, comments, job status — never item bytes |
| **Mixed media sources** | One library index over **local files** and **user clouds** (e.g. Google Drive, Dropbox, pCloud, Google Photos — providers TBD) |
| **Optional local → user-cloud copy** | KinIndex **client** can copy local files into a user-chosen cloud; bytes never transit KinIndex servers |
| **Not a media cloud** | KinIndex is not the storage destination; user’s disk or user’s cloud provider is |
| **Historical-first library** | Years of existing photos/videos across those sources |
| **Direct analysis path** | Client (or provider-accessible path) → model host; KinIndex stores `sourceRef` / `analysisRef` + tags |
| **Who / when / what / where** | Person-first knowledge; activities, time, place as available |
| **Named person consolidation** | Likeness suggestions + **required** human confirm/correct (unlink/split) + naming across the mixed archive |
| **Search & retrieve** | e.g. find all items containing **PersonA** regardless of source |
| **View with overlays** | Open each hit; show media + per-person markers (e.g. colored squares) |
| **Comments** | Auto and manual comments on items (and later video key periods) |
| **Video key periods** | Timeline **ranges** (`start`–`end`) with overlays/comments; person likeness from appearance **within** the period |
| **Batch + honest progress** | Large ingest, copy-to-cloud, and tagging jobs with visible status |
| **Account-first** | Username/password or social (e.g. Google) + 2FA |
| **Private by default** | Per-account isolation; family sharing only if/when specs add it |

**Anchor scenario (end-state):** User searches for PersonA → opens each matching item (wherever that file lives) → views content with overlays and comments → for videos, uses key periods.

**Source reality:** Family media is typically **mixed** (phone cloud libraries + leftover local disks). KinIndex must support that mixture — not assume “all local” or “all cloud.”

---

## Shared constraints

| Constraint | Implication |
|------------|-------------|
| KinIndex **never** reads/stores media | No upload-into-KinIndex; no server-side proxy of item bytes for convenience |
| Local → user-cloud copy | **Client → user’s cloud API only**; service records progress + resulting `sourceRef` |
| Multi-user private **index** | Cloud service owns accounts, item refs, persons, tags, comments, jobs |
| Viewer is a **client** concern | Client resolves media by `sourceType` + refs, then composites overlays from index JSON |
| Search is **index** search | PersonA query hits KinIndex DB; open/view uses the appropriate source connector |

---

## Solution A — Web-heavy

### Idea

Browser is the primary client for almost everything. Desktop optional/absent.

### Strengths / weaknesses (short)

- **Strengths:** Best index-anywhere UX; strong when items already have cloud URLs.  
- **Weaknesses:** Weak at multi-year **local** batch, durable folder access, and local → cloud copy at archive scale.

### End-state fitness

Insufficient alone for mixed sources with serious local leftovers. Fine as *one* surface in a hybrid.

---

## Solution B — Web-light

### Idea

Desktop is the primary client; web is thin (account / metadata).

### Strengths / weaknesses (short)

- **Strengths:** Best local ingest, local → cloud copy, local viewer + overlays.  
- **Weaknesses:** Weak index/view-from-anywhere unless cloud URLs exist and a web client can use them.

### End-state fitness

Strong for local-heavy work; incomplete alone for cloud-primary and multi-device viewing of mixed libraries.

---

## Comparison (A vs B)

| Dimension | Web-heavy | Web-light |
|-----------|-----------|-----------|
| Local batch + local→cloud copy | Weak | Strong |
| Cloud-linked view anywhere | Strong | Needs thin web |
| Mixed library (local + clouds) | Partial | Partial |
| PersonA → view + overlays | Depends on URL availability | Strong on desktop for local |
| KinIndex reads/stores media | Never | Never |

Pure A or pure B each covers only part of a **mixed-source** end state.

---

## Recommended solution — Hybrid (mixed sources)

### Recommendation

**Adopt a hybrid client strategy** as the end-state default:

| Surface | Role |
|---------|------|
| **KinIndex Cloud Service** | Accounts, auth, library **index**, persons/tags/comments, job orchestration — **never media bytes** |
| **Desktop client (required for full product)** | Local library connect, large batch, **local → user-cloud copy**, local viewing + overlays, heavy video key-period review |
| **Web app (first-class, not thin-only)** | Sign-up/2FA, connect user clouds (OAuth), search/browse index, view + overlays when media is URL-accessible, person naming, job status |
| **Mobile (later)** | Search + view for cloud-backed items; optional capture later — still no media stored on KinIndex |

This is **not** “web PoC forever” and **not** “desktop only.” It matches mixed family archives.

### Why hybrid wins for mixed sources

| Need | How hybrid handles it |
|------|------------------------|
| Media already in Drive / Dropbox / pCloud / etc. | Web (and desktop) link provider → index refs → view via provider URLs |
| Media still on disk | Desktop connects folders; can **copy to user cloud** (client-side) and/or analyze from local via client → model host |
| Same PersonA across sources | One cloud index; `personId` links items from any `sourceType` |
| View on another device | Works for **cloud-sourced** (and copied-to-cloud) items in web/mobile; local-only items need the machine that has the files (or copy-to-cloud first) |
| Never become a storage product | Destination of copy is **user’s** cloud; KinIndex only updates index/job rows |

### Media source model (index)

Every item in KinIndex carries source metadata, for example:

| Field | Purpose |
|-------|---------|
| `sourceType` | `local` \| `user_cloud` (and provider id: `google_drive`, `dropbox`, `pcloud`, …) |
| `sourceRef` | Provider file id / path token — **not** file bytes |
| `analysisRef` | Model-accessible reference; **stored in DB** for jobs/retries/future use (ref only, not bytes) |
| `availability` | e.g. `local_only` \| `cloud_backed` \| `analysis_ephemeral` — drives which clients can view |

Search returns items from **all** sources. The client that opens an item uses the right **connector** (local FS vs cloud SDK). If it cannot resolve bytes, it prompts: open on desktop, reconnect cloud, or run **copy to cloud**.

### Local → user-cloud copy (feature)

```
[Desktop KinIndex Client]
    │  reads local files (on device only)
    │  uploads via user’s OAuth to Drive / Dropbox / pCloud / …
    ▼
[User’s cloud storage]  ──►  new sourceRef (cloud)
    │
    └── progress events / completed refs ──► [KinIndex Cloud Service]
                                            (status + index update only;
                                             never receives media bytes)
```

**Rules:**

- Copy is a KinIndex **product feature**, implemented in the **client**  
- KinIndex servers **do not** receive, buffer, or inspect the files  
- After copy, prefer treating the cloud object as the durable `sourceRef` for multi-device view  
- User chooses destination provider/folder; KinIndex does not become that provider  

### Analysis path (unchanged boundary)

- **Cloud-sourced items:** grant/stage so the model can read (provider-specific); KinIndex orchestrates jobs only  
- **Local items:** desktop client → model host direct upload (or copy-to-cloud first, then analyze from cloud)  
- Tags always land in the KinIndex index  

### Viewer path (anchor scenario)

1. Web or desktop: search PersonA (index only).  
2. User opens an item.  
3. Client loads media:
   - **cloud** → provider/stream URL  
   - **local** → desktop file path (web may deep-link “open in desktop app” if local-only)  
4. Client draws overlays + comments from KinIndex JSON.  
5. Videos: **key periods** from index + client player (scrub the span; overlays for persons shown **during** the period).

### Architecture sketch

```
                    ┌─────────────────────────────────────┐
                    │     KinIndex Cloud Service            │
                    │  auth · index · tags · comments · jobs │
                    │     (never user media bytes)          │
                    └──────────────▲────────────────────────┘
                                   │ JSON only
              ┌────────────────────┼────────────────────┐
              │                    │                    │
     ┌────────┴────────┐  ┌────────┴────────┐  ┌───────┴────────┐
     │  Web app        │  │  Desktop client │  │  Mobile later  │
     │  cloud OAuth    │  │  local + copy   │  │  cloud view    │
     │  search / view  │  │  search / view  │  │  search / view │
     └────────┬────────┘  └────────┬────────┘  └───────┬────────┘
              │                    │                    │
              │         ┌──────────┼──────────┐         │
              │         │          │          │         │
              ▼         ▼          ▼          ▼         ▼
        [User clouds: Drive / Dropbox / pCloud / …]   [Local disk]
              │                    │
              └────────┬───────────┘
                       ▼
              [Model host — analysis only]
```

### Capability matrix (recommended hybrid)

| Capability | Desktop | Web | Service |
|------------|---------|-----|---------|
| Connect local folders | Yes | Limited / optional | No media |
| Local → user-cloud copy | **Yes (primary)** | Optional small batches only | Progress/refs only |
| Connect user clouds | Yes | **Yes (primary OAuth)** | Store tokens/refs securely — not files |
| Batch analyze | Yes | Yes (cloud-backed) | Orchestrate only |
| Search PersonA | Yes | Yes | Index query |
| View + overlays (cloud item) | Yes | Yes | Metadata only |
| View + overlays (local-only) | **Yes** | No (prompt desktop or copy) | Metadata only |
| Person confirm / comments | Yes | Yes | Store text/tags |

### Product guidance for users

- **Best multi-device experience:** keep or **copy** media into a linked user cloud, then use web/desktop freely.  
- **Local-only is allowed:** full experience on the desktop that has the files; other devices see index/tags but not pixels until cloud-backed.  
- KinIndex never offers “upload your library to us.”

---

## Sequencing (toward the hybrid)

| Phase | Deliver | Still aim at |
|-------|---------|--------------|
| Early | Web + cloud connector(s) **or** desktop + local — pick one path to prove index + tags + overlays | Same invariant; mixed-source index schema from day one |
| Next | Add the other client surface; ship **local → user-cloud copy** on desktop | Anchor scenario across sources |
| End-state | **Hybrid as above** — mixed sources, shared index, dual clients | Full specs north-star |

Do not encode pure web-heavy or pure web-light as the long-term architecture once mixed sources are required.

---

## Open decisions (within the hybrid)

1. Which user-cloud providers are **launch** vs later (Drive, Google Photos, Dropbox, pCloud, …)?  
2. After local → cloud copy, is local `sourceRef` dropped, kept as secondary, or user-selectable “authoritative” source?  
3. For analysis from Dropbox/pCloud, prefer grant-in-place vs stage to model host?  
4. Minimum web viewing rule: cloud-backed only, or also short-lived analysis URLs?  
5. Desktop technology: Electron, Tauri, or native?

---

## Changelog

| Date | Change |
|------|--------|
| 2026-07-15 | Initial draft: web-heavy vs web-light vs comparison |
| 2026-07-15 | Reframed around **end-state** requirements; PoC treated as sequencing only |
| 2026-07-15 | **Recommended hybrid** for mixed sources; local→user-cloud copy; never read/store media |
| 2026-07-15 | Hard constraints deferred to canonical [`Rules.md`](./Rules.md) |
