# Resume Screening with RAG — Assignment

An AI-powered resume screening tool using **RAG (Retrieval-Augmented Generation)** to analyze resumes and job descriptions, calculate match scores, and enable intelligent Q&A about candidates.

**Tech Stack:** Node.js + Express + TypeScript + React + Vite + OpenAI (embeddings)

**Key Features:**
- Resume + JD upload (PDF/TXT)
- Match score, strengths/gaps analysis
- RAG-powered Q&A (embeddings → retrieval → LLM generation)
- Session persistence
- Sample files included

---

## Architecture

```
┌─────────────────┐
│   React UI      │  (File upload, match display, chat)
│   (Frontend)    │
└────────┬────────┘
         │ HTTP (axios)
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Express Backend                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  POST /api/upload/files                  GET /api/upload/session │
│  ├─ Parse resume/JD (PDF→text)           └─ Retrieve session    │
│  ├─ Chunk text (semantic boundaries)                             │
│  ├─ Embed chunks (OpenAI)                                        │
│  ├─ Upsert to Vector Store                                       │
│  ├─ Calculate match (cosine similarity)                          │
│  └─ Extract fields (email, education, years)                     │
│                                                                   │
│  POST /api/chat/ask                      Vector Store            │
│  ├─ Embed question                        (File-backed)          │
│  ├─ Query vector store (top-k retrieval)                         │
│  ├─ Pass context + question to LLM                               │
│  └─ Return answer                                                │
│                                                                   │
│  Persistent Session Storage (sessions.json)                      │
│  └─ Save embedding vectors, conversation history                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## RAG Flow Diagram

```
1. Upload Resume + JD
   ↓
2. Chunk text into semantic sections
   ↓
3. Generate embeddings (OpenAI)
   ↓
4. Store in Vector DB (file / Pinecone)
   ↓
5. User asks: "Does candidate have React experience?"
   ├─ Embed question (same model)
   ├─ Search vector DB (cosine similarity)
   ├─ Retrieve top-k resume chunks
   ├─ Format context
   │
6. Call LLM with [CONTEXT + QUESTION]
   └─ Return: "Yes, candidate has 5 years React experience..."
```

---

## File Structure

```
Assignment 1/
├── README.md                    ← You are here
├── DEMO.md                      ← Demo walkthrough script
├── docker-compose.yml           ← Docker Compose setup
├── .gitignore                   ← Git ignore patterns
│
├── backend/
│   ├── src/
│   │   ├── index.ts            ← Express server entry
│   │   ├── routes/
│   │   │   ├── upload.ts       ← POST /api/upload/files
│   │   │   └── chat.ts         ← POST /api/chat/ask
│   │   └── services/
│   │       ├── ragService.ts   ← RAG core (embedding, chunking, match, LLM)
│   │       └── vectorStore.ts  ← Vector storage (file/Pinecone)
│   ├── scripts/
│   │   └── pinecone_init.mjs   ← Pinecone init & test script
│   ├── sample_files/           ← Sample resumes & JDs
│   ├── data/                   ← Sessions, embeddings (auto-created)
│   ├── package.json
│   ├── tsconfig.json
│   ├── Dockerfile
│   ├── .env.example
│   └── README.md
│
└── frontend/
    ├── src/
    │   ├── App.tsx            ← Main React component
    │   └── main.tsx           ← Entry point
    ├── index.html
    ├── package.json
    ├── tsconfig.json
    ├── vite.config.ts
    └── README.md
```

---

## API Reference

### Upload & Analyze

**POST `/api/upload/files`**

Upload resume and job description for analysis.

```bash
curl -X POST http://localhost:4000/api/upload/files \
  -F "resume=@resume.pdf" \
  -F "jd=@job_description.txt"
```

Response:
```json
{
  "sessionId": "uuid",
  "analysis": {
    "matchScore": 75,
    "strengths": ["React", "Node.js", "TypeScript"],
    "gaps": ["Kubernetes", "AWS"],
    "fields": {
      "email": "john@example.com",
      "education": "B.S Computer Science",
      "years": 5,
      "skills": ["React", "Node.js", "PostgreSQL", "Docker"]
    },
    "summary": "..."
  }
}
```

**GET `/api/upload/session/:id`**

Retrieve a saved session.

```bash
curl http://localhost:4000/api/upload/session/uuid
```

Response: Full session object with chunks, conversation history, analysis.

### Chat (RAG Q&A)

**POST `/api/chat/ask`**

Ask a question about the resume using RAG.

```bash
curl -X POST http://localhost:4000/api/chat/ask \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "uuid",
    "question": "Does this candidate have experience with React?"
  }'
```

Response:
```json
{
  "answer": "Yes, the candidate has 5 years of React experience with strong TypeScript skills."
}
```

**Note:** The backend:
1. Embeds your question
2. Searches the vector store for relevant resume chunks
3. Passes top chunks + your question to OpenAI LLM
4. Returns the LLM's answer (not a pre-computed response)

---

## Quick start (local)

1. Copy env and set OpenAI key:
```bash
cp "backend/.env.example" "backend/.env"
# edit backend/.env and add OPENAI_API_KEY
```

2. Start services locally (separate terminals):

Backend:
```bash
cd "backend"
npm install
npm run start
```

Frontend:
```bash
cd "frontend"
npm install
npm run dev
```

Done! Open `http://localhost:5173` in your browser.

---

## Demo walkthrough

1. Open the frontend at `http://localhost:5173` (or Vite's printed URL).
2. Upload one resume (`backend/sample_files/resume1.txt`) and one JD (`backend/sample_files/jd1.txt`).
3. The app will show a match score, strengths/gaps, parsed fields, and top resume highlights.
4. Ask questions in the chat — the backend performs RAG: it converts your question into an embedding, retrieves relevant resume chunks, and asks the LLM with the retrieved context.

Architecture (mermaid)

```mermaid
flowchart LR
  A[Upload: resume + JD] --> B[Backend: parse & chunk]
  B --> C[Embeddings (OpenAI)]
  C --> D[Vector Store (file-backed)]
  D --> E[Retrieval on question]
  E --> F[LLM (OpenAI) — answer with context]
  F --> G[Frontend: chat response]
```

Notes on uniqueness and deterministic behavior
- The project mixes rule-based parsing (regex heuristics for email, education, years, skills) with RAG to avoid purely speculative LLM-only parsing.
- Sessions and embeddings are persisted to `backend/data` to enable reproducible demos.
