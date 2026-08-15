## 📝 [Phase 0: Foundation & Understanding](PHASE_0.md) (Weeks 1–2)

**Goal:** Know the codebase cold before changing anything.

- **Read the entire codebase end-to-end.** It's a server, so trace the request lifecycle: image intake → OCR → translation → response. Document the flow in a `ARCHITECTURE.md` in your repo.
- **Map the config system.** There are many `config_*.json` files. Understand how they're loaded, validated, and dispatched. This is likely the first thing you'll refactor for better UX.
- **Get the test suite running.** Understand what `tests/` covers and what it doesn't.
- **Python learning focus:** Package structure (`pyproject.toml`, `src/` layout), virtual environments, imports, basic typing. Cursor is great for "explain this function" queries here.

**Deliverable:** `ARCHITECTURE.md` with a request lifecycle diagram, and a list of pain points you've identified.

---

## Phase 1: Developer Experience & Code Hygiene (Weeks 3–4)

**Goal:** Make the project pleasant to work in — for yourself and future contributors.

- **Linting & formatting:** Set up `ruff` (fast, modern, replaces flake8 + isort + black). Add it to a pre-commit hook.
- **Type checking:** Add `mypy` or `pyright` (Cursor integration is excellent). Start with loose settings and tighten over time.
- **Documentation:** Convert the bilingual README sections into clean, well-structured docs. Consider splitting into a `docs/` tree.
- **CI pipeline:** GitHub Actions workflow that runs tests, linting, and type checks on every push/PR.
- **Python learning focus:** Decorators, context managers, dataclasses/Pydantic models for config validation, `pathlib` for file handling.

**Deliverable:** Green CI, clean lint/type checks, a CONTRIBUTING.md.

---

## Phase 2: Configuration UX Overhaul (Weeks 5–7)

**Goal:** The biggest UX problem right now is configuration. Users have to manually edit JSON files and pick from a dozen example configs.

- **Interactive setup wizard:** A `pgtranslate init` CLI command (using `typer` or `click`) that walks users through provider selection, API key entry, and generates the config automatically.
- **Config schema validation:** Replace raw JSON dicts with Pydantic models. Catch misconfigs at startup, not at runtime. Provide clear error messages.
- **Provider abstraction:** Clean up the backend dispatch so adding a new provider is a single class, not a scattered set of if/else branches. Strategy pattern or plugin registry.
- **Hot-reload config:** Allow changing providers/settings without restarting the server.
- **Python learning focus:** Pydantic, CLI frameworks (typer/click), abstract base classes / protocols, the strategy pattern, Python's `logging` module.

**Deliverable:** `pgtranslate init` wizard, validated config schema, pluggable provider architecture.

---

## Phase 3: Server & API Improvements (Weeks 8–10)

**Goal:** Make the server more robust, observable, and extensible.

- **Async refactor:** If the server is currently synchronous, evaluate moving to `asyncio` + `aiohttp`/`FastAPI`. This matters for latency in real-time game translation.
- **Health & status endpoints:** `/health`, `/status` (current provider, queue depth, uptime), `/providers` (list available backends).
- **Request queuing & rate limiting:** Prevent API cost explosions. Add a simple in-memory or Redis-backed queue.
- **Structured logging:** JSON logs with request IDs for debugging.
- **Python learning focus:** `asyncio`, `async`/`await`, FastAPI or aiohttp, middleware patterns, concurrency primitives.

**Deliverable:** Robust async server with observability endpoints and proper logging.

---

## Phase 4: OCR & Translation Quality (Weeks 11–14)

**Goal:** This is where you start differentiating from both VGTranslate3 and ZTranslate.

- **LLM-powered OCR pipeline:** Instead of just Tesseract → translate, use vision LLMs (GPT-4o, Gemini Vision, local via Ollama) for OCR. They're dramatically better at stylized game fonts.
- **Context-aware translation:** Feed the LLM game context (title, genre, prior translations in session) for more natural translations. Maintain a translation memory per session.
- **Glossary/terminology support:** Let users define terms (character names, item names) that should be translated consistently or left untranslated.
- **Translation caching:** Cache OCR + translation results by image hash to avoid redundant API calls for repeated screens (very common in games).
- **Python learning focus:** Working with REST APIs (`httpx`), caching patterns, prompt engineering in code, session state management.

**Deliverable:** AI-enhanced OCR pipeline with context awareness, glossaries, and caching.

---

## Phase 5: Client & Distribution (Weeks 15–18)

**Goal:** Make PGTranslate installable and usable by non-developers.

- **PyPI package:** `pip install pgtranslate`. Proper packaging with `pyproject.toml`, entry points, optional dependencies (e.g., `pgtranslate[local]` for Ollama support).
- **Docker improvements:** Multi-stage builds, better defaults, docker-compose example.
- **GUI launcher:** A simple system tray app (using `pystray` or a minimal web UI with FastAPI + a frontend) that starts/stops the server, shows status, and lets you switch providers.
- **Python learning focus:** Packaging (setuptools/hatch), entry points, optional dependencies, basic GUI or web frontend.

**Deliverable:** `pip install pgtranslate`, polished Docker setup, optional GUI launcher.

---

## Phase 6: Competitive Differentiation (Long-term)

**Goal:** Position as a genuine ZTranslate competitor.

- **Plugin system:** Third-party OCR/translation providers as installable plugins.
- **Game profile presets:** Pre-configured settings for popular games (e.g., "PS1 JRPG — Japanese → English" with optimized OCR settings).
- **Community translation database:** Users can share and download community-maintained translations (like ZTranslate's packages).
- **Real-time overlay:** Explore a lightweight overlay that displays translations directly on the game window (this is ambitious — may need a separate companion app).
- **Web dashboard:** A browser-based UI for configuration, monitoring, and reviewing translations.
- **Multi-platform client:** Beyond RetroArch — capture from emulators, Steam games, or anything on screen.