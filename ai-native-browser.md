# AI-Native Browser Blueprint (Local + Open Source)

You can think of this as building “Atlas/Perplexity, but local and open-source” on top of a browser shell plus a local-model stack.

## 1) Decide: extend an existing open-source AI browser vs. roll your own

Given your goals (privacy, deep research, coding, actions), the fastest path is usually:

- Fork/extend **BrowserOS** rather than starting from a blank Chromium fork. BrowserOS is already an open-source, agentic browser with automation tools, MCP integrations, and support for local models via Ollama/LM Studio.  
  - https://www.browseros.com/  
  - https://github.com/browseros-ai/BrowserOS
- Use it as your “Atlas-like” base, then wire in stricter local-only policies and your own research/coding UX.

If you want a minimal custom stack instead:

- Build a Chromium-based shell (Electron, Tauri, or Chromium fork) with Chrome DevTools Protocol (CDP) control for navigation/click/type/form fill.
- Add a local LLM layer (Ollama/LM Studio server, or fully in-browser via WebLLM / transformers.js).
- Add a controller/agent service that receives high-level instructions, plans steps, and calls browser/tools.

The rest of this document assumes a BrowserOS-like extension path, but the ideas also apply to a custom shell.

## 2) Core architecture for a local, private AI browser

### 2.1 Browser Shell (UI + Tabs)

Chromium-based UI handles windows, tabs, history, cookies, etc. Expose:

- Page DOM, URL, selection, scroll position.
- Side panel for the AI.
- Control API (CDP) for clicking, typing, and navigation.

### 2.2 Local Model Layer

Two strong options:

- **Local server models** (more flexible, easier multi-agent):
  - Run Ollama or LM Studio locally (Llama 3, Qwen, DeepSeek, etc.).
  - Browser talks to localhost-only HTTP endpoint (OpenAI-compatible).
- **In-browser models** (maximum privacy, no server):
  - WebLLM (`@mlc-ai/web-llm`) or `transformers.js` loading GGUF/ONNX in-browser.
  - Similar to fully in-browser tools where embeddings, generation, and RAG run inside the tab.  
    - [WebLLM local AI example][medium-weblocal]  
    - [TimeCapsule-SLM deep research][timecapsule]

Practical strategy: support both. Use localhost Ollama for heavier tasks and a smaller in-browser model for quick/offline fallback.

### 2.3 Agent & Tooling Layer

This is what makes it Perplexity/Atlas-like:

- Browser tools: `open_url`, `click(selector)`, `type(text, selector)`, `scroll(direction)`, `extract_page({selectors})`, etc.  
  BrowserOS already includes many such tools. https://www.browseros.com/
- Research tools: `search_web(query)`, `parallel_open_and_summarize(urls)`, `cluster_findings`, `rank_sources`.
- File/RAG tools: `ingest_pdf`, `embed_and_store`, `query_corpus`.
- Coding tools: `open_file`, `apply_patch`, `run_tests` via local MCP server or local RPC bridge.

Agent loop: user goal → plan steps → call tools → evaluate → refine.

### 2.4 Data & Memory Layer (Local Only)

- Session memory: conversations, visited URLs, extracted text, code context.
- Local knowledge stores:
  - Research notebook: SQLite/DuckDB + vector index (HNSW-based WASM index, local Qdrant, or transformers.js + IndexedDB/OPFS).
  - Workspace/project RAG corpora.
- Store in local path (e.g., `~/.local-ai-browser`, with platform-specific equivalents on Windows/macOS) or browser storage (IndexedDB/OPFS).
- Provide explicit export/import for portability.

### 2.5 Privacy & Security

- Strict **local-only mode**:
  - Local models only.
  - No telemetry / no remote model calls.
- Capability + policy checks:
  - Classify data before any outgoing action (PII/financial/etc.).
  - Block or require approval when needed.
- Automation permissions:
  - Per-domain permission gates for form submission/click automation.
  - Live action log for transparency.

## 3) Feature set to match Perplexity / Atlas (and go further)

### 3.1 Deep research on the web

Local Perplexity-style “Research Mode”:

1. User enters broad query.
2. Agent runs search.
3. Opens top-N pages in background tabs/headless frames.
4. Extracts and normalizes page content (title, URL, date, cleaned text).
5. Multi-pass reasoning:
   - Per-source summaries + claims + citations.
   - Cross-source synthesis, disagreements, and evidence gaps.
6. Store extracted data + embeddings in a reusable/exportable **research capsule**.

References:

- [TimeCapsule-SLM deep research][timecapsule]

UX:

- Final answer with inline citations.
- Expandable source list.
- Timeline/topic clusters for long investigations.

### 3.2 “Chat with page” and project workspaces

- **Chat with page**:
  - `get_active_page_text()` → chunk/embed → answer with citations.
  - Keep per-tab short-term memory (recent Q&A + extracted sections).
- **Projects/workspaces**:
  - Group tabs, notes, uploaded docs, and research capsules.
  - Each project has:
    - Vector index.
    - Persona/system prompt.
    - Notebook of synthesized outputs.

### 3.3 Coding + local actions

Atlas + Claude Code style, but local:

- Local dev tools daemon (Node/Python/Rust) exposing:
  - `list_files`, `read_file`, `write_file`, `run(cmd)`.
- Expose as MCP server (BrowserOS MCP plumbing can be reused). https://www.browseros.com/
- End-to-end workflows:
  - Open PR, run local tests, cross-check docs, summarize impact.
  - Combine web research + repo analysis (e.g., protocol risk audits).

### 3.4 Offline/cloudless mode

Sigma Eclipse-style cloudless AI browser direction:

- https://www.sigmabrowser.com/blog/private-ai-browser-with-cloudless-llm

Implementation:

- Bundle default GGUF model or provide one-click model download.
- Local PDF pipeline:
  - WASM PDF parser.
  - Local chunking/embedding/vector storage.
  - Local-only QA path.

User modes:

- **Offline-only**: local models only; no non-page network AI calls.
- **Hybrid**: optional external API keys; user-controlled routing by task.

## 4) Concrete implementation blueprint

### 4.1 Base browser

- Fork BrowserOS and run locally. https://github.com/browseros-ai/BrowserOS
- Disable default non-local providers so default path is local-first.

### 4.2 Local model integration

- Add “Local Providers” settings:
  - Ollama URL (`http://127.0.0.1:11434` default).
  - LM Studio/OpenAI-compatible endpoint.
- Add WebLLM “Lite Model” in side panel using a single `MLCEngine` instance with streaming and runtime stats.  
  - [WebLLM local AI example][medium-weblocal]

### 4.3 Deep Research module

- Add `Deep Research` command:
  - Query → search → headless open top URLs.
  - Extract + local-index corpus.
  - Multi-pass report with saved project artifact.
- Use TimeCapsule-SLM style inspiration for local vectors and exportable capsules.  
  - [TimeCapsule-SLM deep research][timecapsule]

### 4.4 Project-based memory

- Add Projects abstraction:
  - Store visited URLs, imported files, research runs, notes.
  - Maintain per-project vector index.
- Retrieval order: project context first, then live web.

### 4.5 Coding + actions

- Build local tools daemon (Node/TypeScript recommended):
  - File read/write.
  - Command execution (`npm test`, `pytest`, `make`, etc.).
- Register as MCP server and connect to agent:
  - Browse docs.
  - Inspect repo files.
  - Propose/apply patches.
  - Re-run tests and summarize.

### 4.6 Privacy + guardrails

- Add “Data Path” display:
  - Model route: local vs web provider.
  - Data flow route: page → local index/model.
- Require explicit consent for:
  - Sending page/context to non-local providers.
  - Website state-changing actions (form submit, checkout, etc.).

---

This approach yields a local, agentic, research-focused browser that feels close to Perplexity/Atlas while preserving self-hosted control and privacy guarantees.

[medium-weblocal]: https://medium.com/@mrmendoza-dev/add-free-local-ai-to-your-web-apps-directly-in-browser-ae0559dc6a02
[timecapsule]: https://www.reddit.com/r/ollama/comments/1lpchao/timecapsuleslm_open_source_ai_deep_research/
