# 🤖 TubeRAG Assistant

> Ask questions about any YouTube video and get answers with precise timestamp citations — powered by RAG, LangGraph, and Groq.

---

## Problem Statement

YouTube videos are rich sources of information, but they're fundamentally unsearchable. If you want to find what was said about a specific topic in a 2-hour lecture, podcast, or tutorial, your only options are to scrub through the timeline manually or scan an auto-generated transcript that has no structure.

This creates real friction:

- **Learners** can't quickly extract key concepts from long educational content
- **Researchers** can't locate specific claims without rewatching entire videos
- **Professionals** waste time hunting through recordings for decisions or context
- **Viewers** lose the ability to verify or revisit what was said and *when*

**TubeRAG Assistant** solves this by turning any YouTube video into a queryable knowledge base. You paste a URL, and the app fetches the transcript, embeds it into a vector store, and lets you have a full conversation about the video's content — with every answer grounded in exact timestamps so you can jump straight to the source.

---

## 🌐 Demo

!**Link:** (https://tuberag-assistant.streamlit.app/)

---

## Features

- 🔗 **Paste any YouTube URL** — transcripts are fetched automatically via the YouTube Transcript API
- 💬 **Conversational Q&A** — multi-turn chat with full memory across the session
- 🕐 **Timestamp-grounded answers** — every paragraph in the response begins with the `[MM:SS - MM:SS]` range it draws from
- 🎨 **Color-coded timestamps** — timestamps are highlighted in blue in the chat UI for quick visual scanning
- 🔄 **Swap videos on the fly** — load a new video at any time; the vector store resets cleanly
- 🧠 **MMR retrieval** — Maximal Marginal Relevance search reduces redundancy and improves answer diversity
- 💾 **Persistent conversation memory** — each session gets a unique thread ID backed by SQLite checkpointing

---

## Tech Stack

| Layer | Technology |
|---|---|
| **UI** | Streamlit |
| **Transcript Fetching** | `youtube-transcript-api`, `langchain-community` YoutubeLoader |
| **Embeddings** | OpenAI `text-embedding-3-small` |
| **Vector Store** | ChromaDB (persisted locally) |
| **LLM** | `gpt-oss-120b` via Groq (fast inference) |
| **Orchestration** | LangGraph `StateGraph` |
| **Memory** | LangGraph `SqliteSaver` (per-thread checkpointing) |
| **Prompt Framework** | LangChain `ChatPromptTemplate` |

---

## How It Works

```
YouTube URL
    │
    ▼
Transcript Fetch (YouTubeTranscriptApi)
    │  Chunks of 25 snippets → Document with [MM:SS - MM:SS] metadata
    ▼
ChromaDB Vector Store
    │  OpenAI text-embedding-3-small
    ▼
LangGraph RAG Pipeline
    │
    ├── [Node 1] Retrieve
    │       MMR search (k=4, fetch_k=20, λ=0.3)
    │       Builds context string with timestamps
    │
    └── [Node 2] Generate
            Groq LLM + strict timestamp-per-paragraph prompt
            Returns answer grounded only in retrieved context
    │
    ▼
Streamlit Chat UI
    Timestamps color-highlighted in blue
    Session memory via SQLite (per thread_id)
```

---

## Project Structure

```
TubeRAG-Assistant/
│
├── app.py                    # Streamlit entry point
│
├── backend/
│   ├── data_loader.py        # Transcript fetching and chunking
│   ├── vectore_store.py      # ChromaDB setup and MMR retriever
│   ├── graph.py              # LangGraph pipeline (retrieve → generate)
│   ├── state.py              # RAGState TypedDict definition
│   ├── prompt.py             # System prompt with timestamp formatting rules
│   └── base_model.py         # LLM init (Groq)
│
├── frontend/
│   ├── sidebar.py            # URL input, transcript loading, video switcher
│   ├── chat_history.py       # Renders persisted chat messages
│   ├── session_state.py      # Streamlit session state initialization
│   ├── page_config.py        # Page title, layout config
│   └── utils.py              # Timestamp colorization (regex → HTML spans)
│
├── architecture/
│   └── graph.png             # LangGraph pipeline diagram
│
├── vector-store/             # ChromaDB persistence (auto-created)
├── backend/chatbot.db        # SQLite checkpoint store (auto-created)
├── requirements.txt
└── pyproject.toml
```

---

## Getting Started

### Prerequisites

- Python 3.11+
- An **OpenAI API key** (for embeddings)
- A **Groq API key** (for LLM inference)

### 1. Clone the repository

```bash
git clone https://github.com/Sharanch3/TubeRAG-Assistant.git
cd TubeRAG-Assistant
```

### 2. Install dependencies

Using `pip`:

```bash
pip install -r requirements.txt
```

Or using `uv` (recommended — a `uv.lock` is included):

```bash
uv sync
```

### 3. Set up environment variables

Create a `.env` file in the project root:

```env
OPENAI_API_KEY=your_openai_api_key_here
GROQ_API_KEY=your_groq_api_key_here
```

### 4. Run the app

```bash
streamlit run app.py
```

Open [http://localhost:8501](http://localhost:8501) in your browser.

---

## Usage

1. Paste a YouTube URL into the sidebar input field
2. Click **Load Transcript** — the app fetches, chunks, and embeds the video transcript
3. Type a question in the chat input (e.g. *"What did they say about transformers?"*)
4. Receive an answer with `[MM:SS - MM:SS]` timestamps — click to jump to that point in the video
5. Continue the conversation — the assistant remembers context across turns
6. Click **Load New Video** to reset and start fresh with a different URL

> **Note:** Only English transcripts are currently supported. The video must have auto-generated or manual English captions enabled.

---

## Environment Variables

| Variable | Description |
|---|---|
| `OPENAI_API_KEY` | Used for generating embeddings via `text-embedding-3-small` |
| `GROQ_API_KEY` | Used for LLM inference via Groq's API |

---

## Design Decisions

**Why MMR retrieval?** Standard similarity search tends to return redundant chunks. Maximal Marginal Relevance balances relevance with diversity (`lambda_mult=0.3`), surfacing a broader range of content from across the video for richer answers.

**Why Groq?** Groq's inference hardware delivers very low latency on open-weight models, making the chat feel responsive even for longer transcripts.

**Why SQLite checkpointing?** LangGraph's `SqliteSaver` persists conversation state per `thread_id` with zero infrastructure — no Redis, no external database. Each new video load gets a fresh UUID thread so history never bleeds across sessions.

**Why chunk at 25 transcript snippets?** YouTube auto-captions produce short ~2–5 second snippets. A 25-snippet chunk covers roughly 60–120 seconds of content — large enough to capture a complete thought, small enough for precise retrieval.

---

## Limitations

- Only supports YouTube videos with English captions
- Answers are limited to what the transcript contains — visual-only content cannot be referenced
- The vector store is single-user and resets when a new video is loaded
- Very long videos (3+ hours) may take longer to embed on first load

---

## License

MIT