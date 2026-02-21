# Resume RAG Backend

Simple TypeScript Express backend implementing RAG for resume screening.

Setup

1. Copy `.env.example` to `.env` and set `OPENAI_API_KEY`.

2. Install deps and run locally:

```bash
cd backend
npm install
# dev server (ts-node)
npm run start
```

3. Build for production (optional):

```bash
npm run build
npm run start:prod
```

API

Endpoints

- POST `/api/upload/files` form-data with `resume` and `jd` files (pdf or txt). Returns `sessionId` and `analysis`.
- GET `/api/upload/session/:id` returns saved session and analysis.
- POST `/api/chat/ask` JSON `{ sessionId, question }` returns `{ answer }`.

Notes

- Sessions and embeddings are persisted to `backend/data/sessions.json` so demo sessions survive a restart.
- The project uses a hybrid approach: deterministic parsing (regex heuristics) + RAG (embeddings + vector search + LLM) for Q&A. This keeps some behavior non-AI deterministic and makes the assignment unique.
- To switch to a managed vector DB (Pinecone/Qdrant), replace the in-memory/file store in `src/services/ragService.ts`.

Troubleshooting

- Make sure `OPENAI_API_KEY` is valid and has embedding/chat quota. If you get errors, check logs printed by the server.

