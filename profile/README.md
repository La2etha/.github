<div align="center">
<img src="logo.png">
# La2etha! · لقيتها

**Found it!** — the exclamation of relief when you finally spot your photo.

*Pool everyone's photos from a gathering, and a custom computer-vision pipeline hands each person a
private gallery of only the shots they're actually in.*

</div>

---

## The problem

At any Egyptian *lamma* or *kherga*, the *shilla* shoots hundreds of photos across a dozen phones and
cameras. When the night ends, getting the right photos to the right people turns into the same tired
chore — scrolling your camera roll, cropping, and firing off *"send me the pics"* a hundred times.
Photos get lost, moments get missed, and nobody ends up with the full set.

**La2etha!** kills that chore. Everyone drops their photos into one shared event. After a quick face
enrollment, each person opens their own gallery and sees **only** the photos they appear in — nothing
else, and nothing of anyone else's.

## What makes it different

This is **not** a thin wrapper over a face-recognition API. It's a purpose-built pipeline engineered
to scale: instead of comparing every registered person against every discovered face (an
`O(people × faces)` explosion that melts a server), we **cluster all faces once at upload time**, then
match each person's identity against a handful of cluster centroids. One clustering pass, one match
per person — the rest is instant.

Access is identity-gated by design: you can only open a source photo you're genuinely in, or one you
own as the event host. Other people's photos are never readable.

## How it works

```mermaid
flowchart LR
    subgraph Clients
        W[React Web]
        M[React Native<br/>later]
    end
    W & M -->|REST / JWT| T[Cloudflare Tunnel]
    T --> API[FastAPI<br/>API + OpenAPI docs]

    API --> DB[(PostgreSQL<br/>+ pgvector)]
    API --> Q[[Redis + RQ<br/>job queue]]
    API --> ST{{Storage interface}}

    Q --> WK[CV Workers<br/>on RTX 2060]
    WK --> DB
    WK --> ST

    ST --> FS[Local FS]
    ST --> R2[Cloudflare R2]
    ST --> GD[Google Drive<br/>ingestion]

    WK -.loads.-> MDL[Models:<br/>SCRFD · ArcFace<br/>SigLIP-2 · LaMa]
```

### The core engine — cluster once, match once

```mermaid
flowchart TD
    U[Photos pooled into event] --> D[Detect faces · SCRFD]
    D --> E[Embed each face · ArcFace-R50<br/>512-d vector]
    E --> C[Cluster anonymously · HDBSCAN<br/>→ one centroid per identity]
    C --> DBc[(Face clusters<br/>+ centroids)]

    EN[User enrolls · 3–5 photos / ~3s video] --> AGG[Aggregate multi-angle<br/>identity centroid]
    AGG --> MATCH{Match centroid<br/>vs cluster centroids}
    DBc --> MATCH
    MATCH --> G[Unlock all photos in<br/>the matched cluster → personal gallery]
```

### A person's journey

```mermaid
sequenceDiagram
    actor Host
    actor Guest
    participant App as La2etha!
    Host->>App: Create event → share join link + code
    Guest->>App: Join event
    Host->>App: Pool photos
    Guest->>App: Pool photos
    App->>App: Detect → embed → cluster (async workers)
    Guest->>App: Enroll face (multi-angle)
    App->>App: Match identity vs cluster centroids
    App-->>Guest: Personal gallery (only their photos)
    Guest->>App: Search "gamb el-torta" / "near the cake"
    App-->>Guest: Matching photos from their gallery
```

## Tech stack

**Backend — [`la2etha-backend`](https://github.com/la2etha/la2etha-backend)**

| Area | Choice |
|------|--------|
| API | Python · FastAPI · auto-generated OpenAPI |
| Auth | FastAPI-Users (JWT + OAuth), self-hosted |
| Data + vectors | PostgreSQL + `pgvector` (one store) |
| Async | Redis + RQ workers |
| Face detection | SCRFD (InsightFace) |
| Face embedding | ArcFace-R50 |
| Clustering | HDBSCAN (benchmarked vs Chinese Whispers, DBSCAN) |
| Quality / proximity | Variance-of-Laplacian + blink + face-scale (Depth-Anything optional) |
| Semantic search | SigLIP-2 (multilingual — handles Arabic queries) |
| Privacy removal | LaMa (local inpainting) |
| AI editing (stretch) | Gemini "Nano Banana" (free tier, opt-in) |

**Frontend — [`la2etha-frontend`](https://github.com/la2etha/la2etha-frontend)**

| Area | Choice |
|------|--------|
| Web | React + TypeScript + Vite |
| Mobile (later) | React Native, reusing the same REST API |
| Hosting | Cloudflare Pages / Vercel → backend via Cloudflare Tunnel |

**Brand palette:** dark teal · soft cream · burnt orange.

## Repositories

- **[`la2etha-backend`](https://github.com/la2etha/la2etha-backend)** — FastAPI service + CV workers. Runs inference locally on a single RTX 2060.
- **[`la2etha-frontend`](https://github.com/la2etha/la2etha-frontend)** — React web app, growing into a React Native mobile app.

## Status

🚧 In active development — a Computer Vision capstone project. The graded centerpiece is the custom
recognition pipeline, evaluated on gallery recall/precision, clustering quality, and a single- vs
multi-angle enrollment ablation against public benchmarks.

## Privacy

Photos and face embeddings stay on our own machine. A person can only see photos they're verified in;
raw pools are never broadly readable. Embeddings and photos are deleted when an event is deleted, and
anyone can delete their own identity and gallery.
