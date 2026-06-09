---
description: Launch the RAG chatbot app and open it in the browser
---

# Run App in Browser

Launches the FastAPI backend, waits for it to be ready, then opens the app at `http://localhost:8000`.

## Prerequisites

- `uv` installed (used for all Python commands — never `pip` or `python` directly)
- `ANTHROPIC_API_KEY` set in `backend/../.env` (i.e. the project root `.env`)

## Run

From the project root, start the server in the background:

```bash
cd backend
uv run uvicorn app:app --reload --port 8000 > /tmp/ragchatbot.log 2>&1 &
echo $! > /tmp/ragchatbot.pid
```

Wait for it to be ready (polls `/api/courses`):

```bash
for i in $(seq 1 30); do
  curl -sf http://localhost:8000/api/courses > /dev/null && echo "Server ready" && break
  sleep 1
done
```

Open in the default browser (Windows):

```bash
start "" "http://localhost:8000"
```

## Verify

```bash
curl -s http://localhost:8000/api/courses
# → {"total_courses":4,"course_titles":[...]}
```

A healthy response lists 4 indexed courses. On first startup the server auto-indexes all `.txt/.pdf/.docx` files in `docs/` — this may take a few seconds longer.

Check logs if the server doesn't come up:

```bash
cat /tmp/ragchatbot.log
```

## Stop

```bash
kill $(cat /tmp/ragchatbot.pid)
# or, if the PID file is gone:
pkill -f "uvicorn app:app"
```

## Notes

- Server listens on port **8000**. If that port is in use, stop the existing process first (`pkill -f "uvicorn app:app"`).
- Hot reload (`--reload`) is active — edits to Python files restart the server automatically.
- ChromaDB data persists in `backend/chroma_db/`. Delete that directory to force a full re-index.
