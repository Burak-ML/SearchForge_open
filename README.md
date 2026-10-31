<p align="center">
  <img src="assets/banner.png" alt="SearchForge banner" width="100%"/>
</p>

# SearchForge

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)
![Next.js](https://img.shields.io/badge/Next.js-black?logo=nextdotjs)

**Full-text search and PDF reading for documents stored in Google Drive.**

Instead of downloading and opening files one by one, you search across a whole Drive folder by file name and by the text inside each PDF, see matching snippets with page numbers, and read the file in the browser.

## Features

- **Full-text search over PDF pages.** Page text is extracted and stored in a local SQLite FTS5 index. Results show a snippet and the page number.
- **Turkish-aware matching.** Text is normalized the same way at index time and at query time, so Turkish characters do not break matches.
- **In-browser reader.** PDFs stream from the backend and open in a PDF.js viewer.
- **Filters.** Narrow results by file type (`pdf`, `docx`, `other`) and by folder.
- **Drive stays the source of truth.** The backend reads Drive through a read-only Service Account, and credentials never reach the frontend. Every file request is checked to be inside the configured Drive folder.
- **Response caching.** Search results are cached with a TTL to avoid repeated work.

## Architecture

```
Next.js frontend  ->  FastAPI backend  ->  Google Drive API
                            |
                     SQLite FTS5 index
                  (page text, built by a sync script)
```

The index stores compressed page text on the machine that runs the backend. Treat that machine as holding a copy of your documents' text.

## Requirements

- Python 3.10+
- Node.js 18+
- A Google Cloud project with a Service Account and a Drive folder shared with it

## Setup

### 1. Google Drive access

1. In the [Google Cloud Console](https://console.cloud.google.com), create or select a project and enable the **Google Drive API**.
2. Create a Service Account (**IAM & Admin → Service Accounts**), then add a JSON key (**Keys → Add Key → Create new key**). No project role is required.
3. In Google Drive, share your source folder with the Service Account's email address with **Viewer** access.
4. Copy the folder ID from the folder URL: `https://drive.google.com/drive/folders/<FOLDER_ID>`.

### 2. Backend

```bash
cd backend
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
```

Edit `.env`:

```env
GOOGLE_DRIVE_FOLDER_ID=your_folder_id_here
GOOGLE_SERVICE_ACCOUNT_FILE=/secure/path/service-account.json
FRONTEND_URL=http://localhost:3000
SEARCH_RESULT_LIMIT=20
MAX_SNIPPETS_PER_FILE=3
CACHE_TTL_SECONDS=300
CACHE_MAX_SIZE=100
SNIPPET_CONTEXT_CHARS=150
```

Keep the Service Account key outside the repository and never commit it or `.env`.

Build the search index, then start the API:

```bash
python -m app.scripts.sync_index --limit 20   # try a small batch first
python -m app.scripts.sync_index              # index everything
python -m app.scripts.sync_index --force      # re-index unchanged files too

uvicorn app.main:app --reload --port 8000
```

Interactive API docs: http://localhost:8000/api/docs

### 3. Frontend

```bash
cd frontend
npm install
cp .env.local.example .env.local
npm run dev
```

`.env.local`:

```env
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_AUTH_USER_HASH=<sha256 of the username>
NEXT_PUBLIC_AUTH_PASS_HASH=<sha256 of the password>
```

Open http://localhost:3000.

## API

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/health` | Service and Drive connection status |
| GET | `/api/search?q=...&type=pdf&folder=...` | Search documents |
| GET | `/api/documents/{file_id}` | Document metadata |
| GET | `/api/documents/{file_id}/content` | PDF stream (inline) |
| GET | `/api/folders` | Folder list |
| GET | `/api/index/status` | State of the search index |

Search parameters: `q` (required), `type` (`all`, `pdf`, `docx`, `other`; default `all`), `folder` (optional path filter).

## Security notes

- **The login screen is not access control.** It compares SHA-256 hashes in the browser, and those hashes are part of the public JavaScript bundle. The backend API has no authentication of its own. Before exposing this to the internet, put real authentication in front of the API, for example a reverse proxy with auth or server-side sessions.
- Never commit the Service Account JSON or `.env` files.
- CORS only allows `FRONTEND_URL`. Do not use a wildcard in production.
- File requests are rejected unless the file is inside the configured Drive folder.
- Add rate limiting (nginx or `fastapi-limiter`) before a public deployment.

## Deployment

```bash
# Backend
pip install gunicorn
gunicorn app.main:app -w 2 -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000

# Frontend
npm run build
```

Set `FRONTEND_URL` to your production domain, put the backend behind a reverse proxy, and set `NEXT_PUBLIC_API_URL` to the backend address.

## Project structure

```
backend/app/
  main.py              FastAPI entry point
  api/                 search, documents and index endpoints
  services/            Drive access, FTS5 index, search, PDF text extraction, cache
  scripts/sync_index.py  builds and refreshes the index from Drive
  core/                settings, validation and error sanitizing
frontend/
  app/                 search page, PDF reader page, login page
  components/          SearchBar, SearchFilters, SearchResult, PdfViewer
  lib/                 API client and the client-side login helper
```

## Ideas for later

- Semantic search with embeddings, combined with the keyword index
- OCR for scanned PDFs
- Server-side authentication and user accounts
- Search history and an admin view for folders and indexing

## License

MIT. See [LICENSE](LICENSE).
