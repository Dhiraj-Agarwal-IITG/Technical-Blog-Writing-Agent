# LangGraph Blog Writing Agent

A Streamlit app that generates a research-backed technical blog post using a
[LangGraph](https://github.com/langchain-ai/langgraph) multi-agent pipeline:
a **router** decides if web research is needed, an **orchestrator** plans the
post into sections, parallel **worker** agents draft each section, and a
**reducer** subgraph merges everything, optionally adds AI-generated images,
and writes the final Markdown file to disk.

---

## Project structure

| File | Purpose |
|---|---|
| `backend.py` | LangGraph pipeline: state schema, router, research, orchestrator, workers, and the image-generation reducer subgraph. Exposes a compiled graph as `app`. |
| `frontend.py` | Streamlit UI that collects a topic, runs `backend.app`, and displays the plan, evidence, markdown preview, images, and logs. |

---

## How it works

```
START
  │
  ▼
router ──(needs_research?)──► research ──► orchestrator
  │                                             │
  └────────────(no research needed)─────────────┘
                                                 │
                                          fanout (Send)
                                                 │
                                     ┌───────────┼───────────┐
                                   worker      worker      worker   (one per planned section)
                                     └───────────┼───────────┘
                                                 ▼
                                     reducer subgraph:
                                     merge_content → decide_images → generate_and_place_images
                                                 │
                                                 END
```

### 1. Router (`router_node`)
Uses `ChatOpenAI` with structured output (`RouterDecision`) to decide:
- Whether web research is needed (`needs_research`)
- The mode: `closed_book` (evergreen, no research), `hybrid` (evergreen + some
  up-to-date research), or `open_book` (news/weekly roundup, volatile topics)
- A recency window (`recency_days`): 7 for open_book, 45 for hybrid, 3650 for
  closed_book
- A list of search queries (if research is needed)

### 2. Research (`research_node`)
If routed to research, runs each query through **Tavily Search**
(`langchain_tavily.TavilySearch`) and asks the LLM to normalize the raw
results into structured `EvidenceItem` objects (title, url, published_at,
snippet, source), deduplicated by URL. For `open_book` mode, evidence older
than the recency cutoff is filtered out. If `TAVILY_API_KEY` is not set,
research silently returns no evidence.

### 3. Orchestrator (`orchestrator_node`)
Produces a structured `Plan` (title, audience, tone, blog kind, constraints,
and 5–9 `Task`s, each with a goal, 3–6 bullets, and a target word count).
`open_book` mode forces `blog_kind="news_roundup"`.

### 4. Fanout (`fanout`)
Uses LangGraph's `Send` API to dispatch one parallel `worker` invocation per
planned task/section, each carrying its own task, the full plan, and the
relevant evidence.

### 5. Worker (`worker_node`)
Each worker asks the LLM to write one Markdown section (`## <Section Title>`)
covering all bullets, respecting the target word count (±15%), and citing
evidence URLs only when required. Sections are collected into `State.sections`
(an `operator.add`-annotated list) so parallel results merge automatically.

### 6. Reducer subgraph (`merge_content` → `decide_images` → `generate_and_place_images`)
- **`merge_content`**: Sorts sections by task id and concatenates them under
  an `# <blog_title>` heading into `merged_md`.
- **`decide_images`**: Asks the LLM to decide if up to 3 diagrams/images would
  materially help the post, inserting `[[IMAGE_1]]`-style placeholders and an
  image prompt/spec for each.
- **`generate_and_place_images`**: For each image spec, generates an image via
  **Google Gemini** (`gemini-2.5-flash-image`, requires `GOOGLE_API_KEY`),
  saves it under `images/`, and replaces the placeholder with a Markdown
  image + caption. If generation fails, the placeholder is replaced with a
  visible "image generation failed" note instead of breaking the document.
  The final Markdown is written to `<slugified-title>.md` in the working
  directory and returned as `State.final`.

### State shape
All nodes read/write a shared `TypedDict` `State`, including: `topic`,
`mode`, `needs_research`, `queries`, `evidence`, `plan`, `as_of`,
`recency_days`, `sections`, `merged_md`, `md_with_placeholders`,
`image_specs`, and `final`.

---

## Frontend (`frontend.py`)

A single-page Streamlit app (`st.set_page_config(layout="wide")`) with:

- **Sidebar**
  - Topic text area and an "As-of date" picker
  - **Generate Blog** button, which builds the initial `State` dict and runs
    `backend.app` via `try_stream` (tries `stream_mode="updates"`, then falls
    back to `stream_mode="values"`, then a plain `invoke`), streaming node
    names and a live JSON progress summary into an `st.status` block
  - **Past blogs**: lists `*.md` files in the working directory (newest
    first), lets you pick one by title/filename and load it back into the
    view (evidence/plan won't be available for files loaded this way, only
    the rendered Markdown)

- **Main area** — five tabs, populated once a result exists in
  `st.session_state["last_out"]`:
  1. **🧩 Plan** — title, audience, tone, blog kind, and a table of planned
     tasks (with a raw JSON expander)
  2. **🔎 Evidence** — table of evidence items used (title, date, source, url)
  3. **📝 Markdown Preview** — renders the final Markdown, resolving local
     `images/...` references (via a custom renderer,
     `render_markdown_with_local_images`, since `st.markdown` alone can't
     display local image files) and remote image URLs, plus **Download
     Markdown** and **Download Bundle (MD + images, zipped)** buttons
  4. **🖼️ Images** — shows the image plan (JSON) and any generated images
     under `images/`, plus a **Download Images (zip)** button
  5. **🧾 Logs** — raw event log of every streamed step (kept in
     `st.session_state["logs"]`, capped to the last 80 entries shown)

---

## Requirements

- Python 3.10+
- An OpenAI API key (`OPENAI_API_KEY`) — required for the router, research
  synthesis, orchestrator, workers, and image-decision steps (all use
  `ChatOpenAI(model="gpt-4.1-mini")`)
- A Tavily API key (`TAVILY_API_KEY`) — optional; without it, research simply
  returns no evidence and the pipeline continues in a closed-book-like manner
- A Google API key (`GOOGLE_API_KEY`) — optional; required only if you want
  the reducer to actually generate images via Gemini. Without it, image
  generation fails gracefully and a placeholder note is inserted instead.

Environment variables are loaded from a `.env` file via `python-dotenv`
(`load_dotenv()` in `backend.py`).

Install dependencies (see `requirements.txt`):

```bash
pip install -r requirements.txt
```

---

## Running the app

1. Create a `.env` file in the project root:

   ```
   OPENAI_API_KEY=sk-...
   TAVILY_API_KEY=tvly-...      # optional
   GOOGLE_API_KEY=...           # optional, needed for image generation
   ```

2. Start the Streamlit app:

   ```bash
   streamlit run frontend.py
   ```

3. Open the app in your browser, enter a topic in the sidebar, and click
   **🚀 Generate Blog**.

Generated posts are saved as `<slugified-title>.md` in the working directory,
and any generated images are saved under `images/`, so both persist between
runs and show up in the **Past blogs** list on subsequent launches.

---

## Notes / known limitations

- Image generation and research are best-effort: missing API keys degrade
  gracefully (no evidence / no images) rather than raising errors.
- Loading a "past blog" from the sidebar only restores the rendered Markdown
  — the original plan and evidence for that run are not persisted to disk,
  so those tabs will show "no plan/evidence found" for older files.
- The app writes files (`*.md`, `images/*`) to whatever directory Streamlit
  is run from — run it from a dedicated project/output folder to keep things
  organized.
