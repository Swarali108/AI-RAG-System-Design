<div align="center">

# 🧠 AI RAG Research Assistant — System Design

### Visual architecture, data flow, and design decisions behind a source-grounded RAG assistant

![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.138-009688?logo=fastapi&logoColor=white)
![Gemini](https://img.shields.io/badge/Google_Gemini-2.5_Flash--Lite-8E75FF?logo=googlegemini&logoColor=white)
![Retrieval](https://img.shields.io/badge/Retrieval-Hybrid_(BM25_%2B_Embeddings)-F06ABF)
![Deploy](https://img.shields.io/badge/Deploy-Vercel-000000?logo=vercel&logoColor=white)

📦 **App repo:** [AI-RAG-Research-Assistant](https://github.com/Swarali603/AI-RAG-Research-Assistant) · 📝 **Case study:** [AI-RAG-Case-Study](https://github.com/Swarali603/AI-RAG-Case-Study)

</div>

---

## 📌 What this repo is

This is the **design companion** to the AI RAG Research Assistant. It explains *how* the system is put together — not the code line-by-line, but the architecture, the flow of a request, and the trade-offs behind each decision — using diagrams that render natively on GitHub.

> **One-line summary:** Upload PDFs → the system extracts, chunks, embeds, and retrieves the most relevant passages with **hybrid (semantic + lexical) search** → Gemini generates a **cited, source-grounded answer**, streamed token-by-token.

---

## 🗺️ 1. System Context

How the pieces relate at the highest level. A single FastAPI service owns everything: it serves the UI, runs the RAG pipeline, and talks to the Gemini API.

```mermaid
flowchart LR
    subgraph Client["🖥️ Browser"]
        UI["Single-page UI<br/>chat + RAG inspector"]
    end

    subgraph Server["⚙️ FastAPI Service (Vercel)"]
        R["Routes<br/>/ · /api/research · /stream"]
        P["RAG Pipeline<br/>parse · chunk · embed · retrieve"]
        C["In-process Cache<br/>per document hash"]
    end

    subgraph Google["☁️ Google Gemini API"]
        E["text-embedding-004"]
        L["gemini-2.5-flash-lite"]
    end

    UI -- "multipart upload + question" --> R
    R --> P
    P <--> C
    P -- "embed chunks / query" --> E
    P -- "generate answer (stream)" --> L
    L -- "Server-Sent Events" --> UI

    classDef client fill:#1f2747,stroke:#65b8ff,color:#d8e6ff
    classDef server fill:#2a1f47,stroke:#9b5cff,color:#e8dcff
    classDef google fill:#3a1f33,stroke:#f06abf,color:#ffd8f0
    class UI client
    class R,P,C server
    class E,L google
```

---

## 🔄 2. RAG Pipeline — Data Flow

The journey of a question, from raw PDF bytes to a cited answer. The **cache check** is the key efficiency win: a document is parsed and embedded **once**, not on every question.

```mermaid
flowchart TD
    A([User uploads PDFs]) --> B[FastAPI /api/research/stream]
    B --> C{Cached?<br/>sha256 of bytes}
    C -- Hit --> H[(Document Index<br/>chunks + embeddings)]
    C -- Miss --> D[pypdf: extract text per page]
    D --> E[Chunk: 220-word windows, 45 overlap]
    E --> F[Embed chunks<br/>Gemini text-embedding-004]
    F --> H
    H --> G[Hybrid retrieval]
    G --> G1[BM25 lexical score]
    G --> G2[Cosine semantic score]
    G1 --> N[Min-max normalize + weighted blend]
    G2 --> N
    N --> K[Top-k chunks + citations]
    K --> L[Build grounded prompt]
    L --> M[Gemini 2.5 Flash-Lite<br/>streamed answer]
    M --> O([SSE: trace / chunk / done])

    classDef entry fill:#9b5cff,stroke:#fff,color:#fff
    classDef store fill:#1f2747,stroke:#65b8ff,color:#d8c3ff
    classDef llm fill:#f06abf,stroke:#fff,color:#fff
    class A,O entry
    class H store
    class F,M llm
```

---

## ⏱️ 3. Request Sequence

What happens on the wire for a single question, including the streaming response.

```mermaid
sequenceDiagram
    actor U as User
    participant FE as Browser UI
    participant API as FastAPI
    participant Cache as Doc Cache
    participant G as Gemini API

    U->>FE: Upload PDF + ask question
    FE->>API: POST /api/research/stream (multipart)
    API->>Cache: lookup sha256(file bytes)
    alt cache miss
        API->>API: extract + chunk PDF
        API->>G: embed chunks (text-embedding-004)
        G-->>API: chunk vectors
        API->>Cache: store index
    end
    API->>G: embed query
    G-->>API: query vector
    API->>API: hybrid rank (BM25 + cosine)
    API-->>FE: SSE event: trace (chunks, scores, prompt)
    API->>G: generate_content_stream(prompt)
    loop streamed tokens
        G-->>API: token
        API-->>FE: SSE event: chunk
    end
    API-->>FE: SSE event: done
    FE-->>U: Animated answer + citations + RAG inspector
```

---

## 🎯 4. Hybrid Retrieval — The Core Idea

Pure keyword search misses synonyms ("downsides" vs "limitations"); pure embeddings can drift on exact terms and names. **Hybrid** retrieval blends both and degrades gracefully.

```mermaid
flowchart TD
    Q([Standalone query]) --> L1[Tokenize: lowercase, drop stopwords]
    Q --> S1[Embed query<br/>RETRIEVAL_QUERY]

    L1 --> L2[BM25 over all chunks<br/>tf saturation + length norm]
    S1 --> S2[Cosine vs cached chunk vectors]

    L2 --> NRM[Min-max normalize each to 0..1]
    S2 --> NRM
    NRM --> W["Blend: 0.6·semantic + 0.4·lexical"]
    W --> T[Sort → top-k]

    S1 -. "embeddings unavailable / no key" .-> FB[BM25-only fallback]
    FB --> T

    classDef q fill:#9b5cff,stroke:#fff,color:#fff
    classDef lex fill:#1f3a2e,stroke:#60f3a9,color:#d8ffe9
    classDef sem fill:#1f2747,stroke:#65b8ff,color:#d8e6ff
    classDef fb fill:#3a2a1f,stroke:#fbbf24,color:#ffe9c8
    class Q,T q
    class L1,L2 lex
    class S1,S2 sem
    class FB fb
```

> **Why it never breaks:** if the embedding call fails (bad key, rate limit, offline), the embedder returns `None` and the ranker silently falls back to BM25-only. The app always answers.

---

## 🧩 5. Component Breakdown

| Layer | Responsibility | Key choices |
| --- | --- | --- |
| **UI** (served HTML/JS) | Upload, chat, animated streaming, RAG inspector | Zero build step — backend serves one page; SSE for streaming |
| **API** (FastAPI) | Routing, validation, CORS, upload limits | Configurable origins, per-file size cap, SSE responses |
| **Ingestion** | PDF → text → chunks | `pypdf`, 220-word windows with 45-word overlap |
| **Embedding** | Chunk & query vectors | Gemini `text-embedding-004`, task-typed (doc vs query) |
| **Cache** | Avoid re-work | `sha256(file bytes)` → index; FIFO-bounded |
| **Retrieval** | Rank relevant chunks | Hybrid BM25 + cosine, normalized blend, top-k |
| **Generation** | Grounded, cited answer | `gemini-2.5-flash-lite`, streamed; Research / Bestie tone |

---

## ⚖️ 6. Design Decisions & Trade-offs

| Decision | Why | Trade-off accepted |
| --- | --- | --- |
| **Single self-contained FastAPI app** | Simple to deploy on serverless (Vercel); no separate frontend build | Less separation than a full SPA + API split |
| **Gemini embeddings instead of local `sentence-transformers` + FAISS** | Serverless-friendly: no heavy model download or native FAISS build | Per-document embedding API call (mitigated by caching) |
| **Hybrid retrieval with BM25 fallback** | Best of semantic + lexical; never fully breaks | Slightly more ranking logic to maintain |
| **Per-request multipart upload** | Stateless, no storage layer to manage | Same file re-sent each request (cache softens cost) |
| **Streaming via SSE** | Fast perceived response; user sees tokens immediately | Slightly more client-side parsing logic |

---

## 🚀 7. Tech Stack

`Python 3.12` · `FastAPI` · `Uvicorn` · `Google Gemini (google-genai)` · `pypdf` · custom `BM25` · `pytest` · `GitHub Actions` · `Vercel`

---

<div align="center">

*Diagrams are written in [Mermaid](https://mermaid.js.org/) and render natively on GitHub.*

</div>
