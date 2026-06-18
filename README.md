# RepoChat-AI

RepoChat-AI is a small Flask web app that lets you point an AI at any codebase — either by uploading a ZIP file or pasting a public GitHub URL — and then chat with it. It walks the source tree, builds a semantic search index over the code, and uses that index to generate a structured project summary or answer free-form questions about the repository.

## Features

- **Two ways to load a repo** — upload a `.zip` (up to 100 MB) or paste a public GitHub URL to clone.
- **AI-generated project summary** — overview, tech stack, architecture, usage instructions, and notable features, generated in one click.
- **Chat with your codebase** — ask questions about architecture, specific functions, bugs, or anything else; answers are grounded in the actual repository content.
- **Semantic search over source files** — repository text is chunked and embedded so the most relevant pieces of code are retrieved for every question, rather than dumping the whole repo into the prompt.
- **Broad language support** — the file scanner recognizes dozens of source file extensions (Python, JS/TS, Java, C/C++, Go, Rust, Ruby, PHP, Swift, Kotlin, and many more) and skips noise directories like `node_modules`, `.git`, `venv`, `dist`, and `build`.
- **Simple, single-page UI** — a Bootstrap-based interface with a sidebar for loading repositories and a main panel for the summary and chat.

## How it works

1. **Ingest** — `services/zip_loader.py` extracts an uploaded ZIP, or `services/github_loader.py` shallow-clones a GitHub URL into a working folder.
2. **Parse** — `services/parser.py` walks the extracted/cloned folder, keeps only recognized source-code extensions, and concatenates each file's contents (truncated per file) into one large "repository context" string.
3. **Index** — `services/embeddings.py` splits that context into fixed-size chunks, embeds them with the `all-MiniLM-L6-v2` sentence-transformers model, and stores the vectors in an in-memory FAISS index.
4. **Ask / Summarize** — `services/chat_engine.py` takes a user question (or a fixed summary prompt), retrieves the most relevant chunks via FAISS similarity search, and sends them as context to an LLM through the Hugging Face Inference API (`services/hf_client.py`).
5. **Serve** — `app.py` exposes this pipeline as a small set of Flask routes, and `templates/index.html` + `static/app.js` provide the browser UI.

## Tech stack

- **Backend:** Python, Flask, Werkzeug
- **Repository ingestion:** GitPython (`gitpython`), Python's built-in `zipfile`
- **Embeddings & search:** `sentence-transformers` (`all-MiniLM-L6-v2`), `faiss-cpu`, NumPy
- **LLM inference:** `huggingface_hub` `InferenceClient`, default model `Qwen/Qwen2.5-Coder-7B-Instruct`
- **Frontend:** Bootstrap 5, Bootstrap Icons, vanilla JavaScript

## Project structure

```
RepoChat-AI/
├── app.py                     # Flask app and routes
├── requirements.txt
├── services/
│   ├── github_loader.py       # Clones public GitHub repos
│   ├── zip_loader.py          # Extracts uploaded ZIP files
│   ├── parser.py              # Walks repo, builds the text context
│   ├── embeddings.py          # Chunking, embeddings, FAISS index/search
│   ├── chat_engine.py         # Question answering + summary generation
│   └── hf_client.py           # Hugging Face Inference API wrapper
├── templates/
│   └── index.html             # Single-page UI
└── static/
    ├── app.js                 # Upload, chat, and UI logic
    └── style.css
```

## Setup

### Prerequisites

- Python 3.10+
- A free [Hugging Face](https://huggingface.co/settings/tokens) account and API token
- `git` installed on your system (used to clone GitHub repos)

### Installation

```bash
git clone <this-repo-url>
cd RepoChat-AI
python -m venv venv
source venv/bin/activate   # on Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### Configuration

Create a `.env` file in the project root:

```env
HF_API_KEY=your_huggingface_api_token
HF_MODEL=Qwen/Qwen2.5-Coder-7B-Instruct
```

`HF_MODEL` is optional and defaults to `Qwen/Qwen2.5-Coder-7B-Instruct` if not set. Without a valid `HF_API_KEY`, repository loading and indexing still work, but chat and summary requests will return a warning instead of an AI-generated answer.

> **Security note:** never commit your `.env` file or share your Hugging Face token. If a token has been exposed (for example, committed to a repo or shared in a chat), revoke it from your [Hugging Face token settings](https://huggingface.co/settings/tokens) and generate a new one.

### Running the app

```bash
python app.py
```

The app starts in debug mode on `http://127.0.0.1:5000` by default.

## Usage

1. Open the app in your browser.
2. In the sidebar, either drag-and-drop / browse for a `.zip` archive and click **Analyze ZIP**, or paste a public GitHub URL and click **Analyze GitHub Repo**.
3. Once indexing finishes, an AI-generated summary appears in the main panel (overview, tech stack, architecture, usage, notable features).
4. Use the chat box to ask questions about the codebase — e.g. "What does the `parser.py` module do?" or "Is there any authentication logic in this project?"

## API reference

| Method | Route               | Description                                                        |
|--------|---------------------|----------------------------------------------------------------------|
| GET    | `/`                  | Serves the web UI                                                     |
| GET    | `/status`            | Returns whether a repository is currently loaded                      |
| POST   | `/upload-zip`        | Accepts a `.zip` file upload, extracts and indexes it                 |
| POST   | `/analyze-github`    | Accepts `{"url": "..."}`, clones and indexes a public GitHub repo     |
| POST   | `/generate-summary`  | Generates a structured AI summary of the currently loaded repository |
| POST   | `/chat`              | Accepts `{"question": "..."}` and returns an AI-generated answer      |

## Limitations

- Repository state is held in memory and is shared across all visitors of a running instance — this is intended for local/single-user use, not multi-tenant production deployment.
- Only one repository can be loaded at a time; loading a new one replaces the previous index.
- Each file's content is truncated to the first 3,000 characters before indexing, so very large individual files will only be partially represented.
- Answers depend on the configured Hugging Face model's availability and free-tier rate limits.

## License

No license file is included in this project. Add one (e.g. MIT, Apache-2.0) if you intend to share or distribute this code.
