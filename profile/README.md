
<div align="center">

<img src="logo.png" width="200" alt="Lahza logo" />

# Lahza · لحظة

**Moment** — because every moment deserves to find its people.

*Pool everyone's photos from a gathering, and a custom computer-vision pipeline hands each person a private gallery of only the shots they're actually in.*

![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=flat&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat&logo=fastapi&logoColor=white)
![PyTorch](https://img.shields.io/badge/PyTorch-EE4C2C?style=flat&logo=pytorch&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL_+_pgvector-4169E1?style=flat&logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat&logo=redis&logoColor=white)
![React](https://img.shields.io/badge/React-61DAFB?style=flat&logo=react&logoColor=black)
![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=flat)

[Problem](#-the-problem) · [What's different](#-what-makes-it-different) · [How it works](#-how-it-works) · [Core engine](#-the-core-engine--cluster-once-match-once) · [Tech stack](#-tech-stack) · [Repositories](#-repositories) · [Team](#-team)

</div>

---

> [!NOTE]
> **Privacy is the whole point.** Photos and face embeddings stay on our own machine. A person can only see photos they're verified in — raw pools are never broadly readable. Delete an event and its photos and embeddings go with it; anyone can delete their own identity and gallery at any time.

---

## 📦 Repositories

| Component | Repository | What's inside |
|-----------|------------|---------------|
| 🧠 CV Pipeline + FastAPI Backend | https://github.com/la2etha/la2etha-backend | Face detection, embedding, clustering, enrollment matching, quality culling, semantic search, privacy export, REST API + async workers |
| 💻 React Frontend | https://github.com/la2etha/la2etha-frontend | Conversational photo-sharing UI — create/join events, pool photos, enroll your face, browse your private gallery |

The backend runs the entire computer-vision pipeline locally on a single GPU. The frontend is a React web app (growing toward React Native mobile) that talks to the FastAPI backend over a REST/JWT API.

---

## 🎯 The Problem

At any Egyptian *lamma* or *kherga*, the *shilla* shoots hundreds of photos across a dozen phones and cameras. When the night ends, getting the right photos to the right people turns into the same tired chore — scrolling your camera roll, cropping, and firing off *"send me the pics"* a hundred times. Photos get lost, moments get missed, and nobody ends up with the full set.

**Lahza** kills that chore. Everyone drops their photos into one shared event. After a quick face enrollment, each person opens their own gallery and sees **only** the photos they appear in — nothing else, and nothing of anyone else's.

<div align="center"><img src="demo.gif" width="700" alt="Lahza demo"/></div>

---

## ✨ What Makes It Different

This is **not** a thin wrapper over a face-recognition API. It's a purpose-built pipeline engineered to scale: instead of comparing every registered person against every discovered face (an `O(people × faces)` explosion that melts a server), we **cluster all faces once at upload time**, then match each person's identity against a handful of cluster centroids. One clustering pass, one match per person — the rest is instant.

Access is identity-gated by design: you can only open a source photo you're genuinely in, or one you own as the event host. Other people's photos are never readable.

---

## 🏗 How It Works

A message from any client hits the FastAPI service over REST/JWT; heavy CV work is offloaded to Redis/RQ workers on the GPU box, and everything persists to one Postgres + `pgvector` store behind a pluggable storage interface.

```mermaid
flowchart LR
    subgraph Clients
        W[React Web]:::pill
        M[React Native<br/>later]:::pill
    end
    W & M -->|REST / JWT| T[Cloudflare Tunnel]:::io
    T --> API[FastAPI<br/>API + OpenAPI docs]:::svc

    API --> DB[(PostgreSQL<br/>+ pgvector)]:::data
    API --> Q[[Redis + RQ<br/>job queue]]:::svc
    API --> ST{{Storage interface}}:::io

    Q --> WK[CV Workers<br/>on RTX 2060]:::core
    WK --> DB
    WK --> ST

    ST --> FS[Local FS]:::data
    ST --> R2[Cloudflare R2]:::data
    ST --> GD[Google Drive<br/>ingestion]:::data

    WK -.loads.-> MDL[Models:<br/>SCRFD · ArcFace<br/>SigLIP-2 · LaMa]:::core

    classDef pill fill:#f5ecd7,stroke:#888,color:#3a3a3a
    classDef svc fill:#ccece8,stroke:#0f766e,color:#134e4a
    classDef data fill:#e6f2f0,stroke:#0d9488,color:#134e4a
    classDef io fill:#faf3e0,stroke:#b45309,color:#7c2d12
    classDef core fill:#fde3d1,stroke:#c2560c,color:#7c2d12
```

### 🔩 The Core Engine — cluster once, match once

The graded, load-bearing part of the system. Faces are detected, embedded, and clustered **anonymously** at upload time — one centroid per identity. Enrollment builds a multi-angle centroid for a person and matches it against those cluster centroids, unlocking a whole cluster of photos in a single comparison.

```mermaid
flowchart TD
    U[Photos pooled into event]:::pill --> D[Detect faces · SCRFD]:::core
    D --> E[Embed each face · ArcFace-R50<br/>512-d vector]:::core
    E --> C[Cluster anonymously · HDBSCAN<br/>→ one centroid per identity]:::core
    C --> DBc[(Face clusters<br/>+ centroids)]:::data

    EN[User enrolls · 3–5 photos / ~3s video]:::pill --> AGG[Aggregate multi-angle<br/>identity centroid]:::core
    AGG --> MATCH{Match centroid<br/>vs cluster centroids}:::io
    DBc --> MATCH
    MATCH --> G[Unlock all photos in the<br/>matched cluster → personal gallery]:::pill

    classDef pill fill:#f5ecd7,stroke:#888,color:#3a3a3a
    classDef data fill:#e6f2f0,stroke:#0d9488,color:#134e4a
    classDef io fill:#faf3e0,stroke:#b45309,color:#7c2d12
    classDef core fill:#fde3d1,stroke:#c2560c,color:#7c2d12
```

<details>
<summary><b>🔎 A person's journey — end to end</b></summary>

<br/>

From event creation to a personalized, searchable gallery. Detection, embedding, and clustering happen asynchronously in the workers, so pooling never blocks on the GPU.

```mermaid
sequenceDiagram
    actor Host
    actor Guest
    participant App as Lahza
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

</details>

---

## 🧩 The Pipeline at a Glance

| Stage | What it does | Approach |
|-------|--------------|----------|
| **Detect** | Find every face in every pooled photo | SCRFD (InsightFace) |
| **Embed** | Turn each face into a 512-d vector | ArcFace-R50 |
| **Cluster** | Group faces into anonymous identities | HDBSCAN (vs Chinese Whispers, DBSCAN) |
| **Enroll & match** | Build a multi-angle identity, match to clusters | Centroid aggregation + centroid match |
| **Quality cull** | Demote blurry / blinking / tiny-face shots | Variance-of-Laplacian + blink + face-scale |
| **Proximity** | Demote photos where you're a distant bystander | bbox-ratio + face-sharpness (Depth-Anything optional) |
| **Search** | Natural-language photo search, incl. Arabic | SigLIP-2 (multilingual) |
| **Privacy export** | Remove strangers' faces from a shared photo | LaMa local inpainting |
| **AI edit** *(stretch)* | Prompt-based edit of your own solo photos | Gemini "Nano Banana" (opt-in) |

> Quality and proximity are **non-destructive** — low-relevance shots are demoted to a secondary "you're probably not interested in these" section, never hidden or deleted. The host can promote anything back.

---

## 🛠 Tech Stack

### Backend — [`la2etha-backend`](https://github.com/la2etha/la2etha-backend)

![Python](https://img.shields.io/badge/-Python_3.11+-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/-FastAPI-009688?logo=fastapi&logoColor=white)
![PyTorch](https://img.shields.io/badge/-PyTorch-EE4C2C?logo=pytorch&logoColor=white)
![InsightFace](https://img.shields.io/badge/-InsightFace-5C3EE8?logoColor=white)
![PostgreSQL](https://img.shields.io/badge/-PostgreSQL-4169E1?logo=postgresql&logoColor=white)
![pgvector](https://img.shields.io/badge/-pgvector-4169E1?logoColor=white)
![Redis](https://img.shields.io/badge/-Redis-DC382D?logo=redis&logoColor=white)
![SQLAlchemy](https://img.shields.io/badge/-SQLAlchemy-D71F00?logo=sqlalchemy&logoColor=white)
![scikit-learn](https://img.shields.io/badge/-scikit--learn-F7931E?logo=scikit-learn&logoColor=white)

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

### Frontend — [`la2etha-frontend`](https://github.com/la2etha/la2etha-frontend)

![React](https://img.shields.io/badge/-React-61DAFB?logo=react&logoColor=black)
![TypeScript](https://img.shields.io/badge/-TypeScript-3178C6?logo=typescript&logoColor=white)
![Vite](https://img.shields.io/badge/-Vite-646CFF?logo=vite&logoColor=white)
![React Router](https://img.shields.io/badge/-React_Router-CA4245?logo=reactrouter&logoColor=white)
![TanStack Query](https://img.shields.io/badge/-TanStack_Query-FF4154?logo=reactquery&logoColor=white)

| Area | Choice |
|------|--------|
| Web | React + TypeScript + Vite |
| Mobile (later) | React Native, reusing the same REST API |
| Hosting | Cloudflare Pages / Vercel → backend via Cloudflare Tunnel |

**Brand palette:** 🟢 dark teal · 🟡 soft cream · 🟠 burnt orange.

---

## 🔒 Privacy by Design

- **Local first.** Photos and face embeddings live on our own machine; inference runs on a single RTX 2060, not a third-party API.
- **Identity-gated access.** Every photo read resolves through a single access guard — you can open a photo only if you're verified in it or you host the event. Otherwise it returns **404**, never even revealing the photo exists.
- **Deletable by design.** Deleting an event cascades to its stored bytes and embeddings; anyone can delete their own identity and gallery.

---

## 👥 Team

| Ahmed El Sayed | Mohamed Emad | Ziad Mahmoud |
|:---:|:---:|:---:|
| [![GitHub](https://img.shields.io/badge/-GitHub-181717?logo=github)](https://github.com/ahmed-elsayid) | [![GitHub](https://img.shields.io/badge/-GitHub-181717?logo=github)](https://github.com/3omdawy11) | [![GitHub](https://img.shields.io/badge/-GitHub-181717?logo=github)](https://github.com/ZeyadMahmoudAmrMohamed) |

<div align="center">
<br/>
<sub>Lahza · لحظة — every moment deserves to find its people.</sub>
</div>
