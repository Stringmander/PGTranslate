# Phase 0: Foundation & Understanding (Weeks 1–2)

## Primary Goals

1. **Internal mental model:** Be able to trace any request from receipt to response without looking up code.
2. **Pain point inventory:** Document UX issues, architectural smells, and technical debt before refactoring.
3. **Learning baseline:** Understand enough Python idioms to recognize when something is "off."

## Success Criteria

By the end of Phase 0, you should be able to:

- Draw the request flow on paper from memory
- Locate and modify any config type without breaking the server
- Explain what each major file/module does in one sentence
- Confidently predict where a bug would manifest given a symptom

## Deliverables Directory Structure

```text
docs/
├── roadmap/
│   ├── roadmap.md                     # High-level project trajectory (all phases)
│   └── phase_0.md                     # This document
└── deliverables/
    ├── PHASE_0_NOTES.md               # First-pass readings, confusion points
    ├── ARCHITECTURE_DRAFT.md          # Code modules, request lifecycle, data flow
    ├── DOCUMENTATION_MAP.md           # Existing docs inventory + gaps
    ├── CONFIG_SYSTEM_DRAFT.md         # Config provider mappings, validation gaps
    ├── RETROARCH_INTEGRATION.md       # Protocol details, request/response schemas
    └── PHASE_0_SUMMARY.md             # Final synthesis + priority-ranked pain points
```

### Naming Convention

All `*_DRAFT.md` files are promoted to their final form (dropping the `_DRAFT` suffix) upon Phase 0 completion via `git mv`:

```bash
git mv docs/deliverables/ARCHITECTURE_DRAFT.md docs/deliverables/ARCHITECTURE.md
git mv docs/deliverables/CONFIG_SYSTEM_DRAFT.md docs/deliverables/CONFIG_SYSTEM.md
git commit -m "docs: promote Phase 0 drafts to final form"
```

Files without `_DRAFT` (`PHASE_0_NOTES.md`, `DOCUMENTATION_MAP.md`, `RETROARCH_INTEGRATION.md`, `PHASE_0_SUMMARY.md`) are created in their final name from the start.

Phase-scoped deliverables carry the phase number in `PHASE_<n>_` form, matching the `phase_<n>.md` naming used under `docs/roadmap/`.

---

## Week 1: Codebase Reconnaissance

### Day 1–2: First Pass Reading

**Task:** Read files top-to-bottom in this order:

1. `README.md` — Get the high-level pitch
2. `INSTALL.md` — Understand dependencies and setup
3. `LOCAL_MODELS_GUIDE.md` and `TESSERACT_GUIDE.md` — Specialized setup docs
4. `src/vgtranslate3/` directory — Read all `.py` files
5. `src/vgtranslate3/config_example/` — The ten `config_*.json` provider examples
6. `tests/` — Skim what's tested vs. not tested
7. `pyproject.toml` and `requirements.txt` — Dependencies

**Deliverable:** Create `docs/deliverables/PHASE_0_NOTES.md` with:

- One sentence describing each file/module's purpose
- A list of 5–10 terms/concepts you had to look up
- Your initial gut feelings on what feels clean vs. messy
- Red flags encountered (hardcoded paths, silent failures, monolithic functions, magic numbers, global state — see "Red Flags to Watch For" below)

**Deliverable:** Create `docs/deliverables/DOCUMENTATION_MAP.md` with:

- A table mapping each doc file to its purpose, target audience, and completeness assessment
- Gaps in documentation (e.g., "no troubleshooting guide," "INSTALL.md assumes Linux only")
- Docs that are stale, duplicated, or contradictory

**Cursor usage tip:** For any function/file you don't understand, highlight the code and ask:

> "Explain this function's purpose, inputs, outputs, and side effects. What's a realistic scenario where this would fail?"

Don't rush. Spend 30–60 minutes per file if needed. The goal is comprehension, not speed.

### Day 3–4: Trace the Request Lifecycle

**Task:** Manually trace an image request from arrival to translation completion.

Start at `src/vgtranslate3/serve.py`, which `pyproject.toml` exposes as the `vgtranslate3` console script via `vgtranslate3.serve:main`. Note that `main` is a function inside `serve.py` — there is no `main.py` module. From there, follow:

- How does the image get received?
- Where does OCR happen?
- Where does translation happen?
- How are errors handled at each stage?
- What gets returned to the client?

**Deliverable:** Create `docs/deliverables/ARCHITECTURE_DRAFT.md` with the following structure (fill in as you discover answers — the template guides you toward what matters most):

**One-Sentence Summary:** PGTranslate is a [one-sentence description of what the project does].

**Request Lifecycle Diagram:** A text-based diagram showing how a game screenshot flows from entry to translation output.

**Module Breakdown** (17 Python files in `src/vgtranslate3/`, ~5,200 lines total). Start with the largest, since they carry most of the logic:

|File/Module|Responsibility|Dependencies|Notes|
|---|---|---|---|
|`serve.py` (entry point)|...|...|...|
|`util.py`|...|...|...|
|`ocr_providers.py`|...|...|...|
|`translation_providers.py`|...|...|...|
|`ocr_tools.py`|...|...|...|
|`imaging.py`|...|...|...|
|`config.py`|...|...|...|
|`__init__.py`|...|...|...|
|...remaining modules...|...|...|...|

**Data Flow Transformation Points:**

|Stage|Input Format|Output Format|Transform|
|---|---|---|---|
|HTTP Receipt|...|...|...|
|Preprocessing|...|...|...|
|OCR|...|...|...|
|Translation|...|...|...|
|Response|...|...|...|

Include an "Unknowns & Questions" section with checkbox items, and a "Notes from Reading" section for observations as you read through each module.

Mark the top of the file with:

> Working document — Phase 0 WIP. Will be promoted to ARCHITECTURE.md upon phase completion.

### Day 5–7: Static Analysis

**Task:** Run the existing tests and examine coverage. The project uses pytest, configured under `[tool.pytest.ini_options]` in `pyproject.toml`.

```bash
uv sync --extra dev          # installs pytest, ruff, mypy
uv pip install pytest-cov    # not yet declared in the dev extra
uv run pytest --cov=src/ --cov-report=html
```

The `--cov` flag comes from the `pytest-cov` plugin, which the `dev` extra does not currently list; without it pytest exits on an unrecognized argument. Adding it permanently is a Phase 1 task. Look at:

- Which modules have zero test coverage?
- Which parts of the code aren't exercised?
- Are there obvious gaps (e.g., error handling paths)?

Also check:

- Static analysis warnings with `uv run ruff check src/ tests/` (config already exists under `[tool.ruff]`)
- Type hints: are they present, partial, or absent?
- Import cycles or circular dependencies

**Deliverable:** Append a "Static Analysis" section to `PHASE_0_NOTES.md` covering:

- Coverage % estimate by module
- List of untested critical paths
- Ruff issues noticed (even if not fixing yet)

---

## Week 2: Configuration & Integration Deep Dive

### Day 8–10: Config System Mapping

**Task:** Understand every config variant in detail.

Look at all `config_*.json` files in `src/vgtranslate3/config_example/` and catalog:

- What provider does each support?
- What fields are common vs. unique?
- How does the code pick which config to load?
- What happens when a required field is missing?

**Deliverable:** Create `docs/deliverables/CONFIG_SYSTEM_DRAFT.md` with:

- A table mapping config files to providers and key fields
- A diagram showing how config selection happens at runtime
- A list of validation gaps (fields that aren't checked, ambiguous defaults)
- Three hypothetical misconfigurations and where the error would surface

### Day 11–12: RetroArch Integration Flow

**Task:** Understand how RetroArch talks to PGTranslate.

Trace:

- What protocol/port does RetroArch use?
- What request format does it expect?
- What response format must PGTranslate return?
- Are there timing constraints (game translation needs to be fast)?

**Deliverable:** Create `docs/deliverables/RETROARCH_INTEGRATION.md` covering:

- Protocol details (HTTP, WebSocket, etc.)
- Expected request/response schemas
- Any latency notes or performance considerations

**Research tip:** If the code doesn't clarify this, search RetroArch AI Service documentation. This is external knowledge you'll need.

### Day 13–14: Compile Findings & Identify Pain Points

**Task:** Synthesize everything into an actionable inventory.

Review your notes from Weeks 1–2 and identify:

1. **UX pain points:** Things users struggle with (e.g., "must manually edit 10 JSON files")
2. **Architecture smells:** Design patterns that feel fragile (e.g., "if/else chain for provider routing")
3. **Technical debt:** Missing tests, unclear error messages, undocumented functions
4. **Quick wins:** Small fixes that would yield big UX gains

**Deliverable:** Create `docs/deliverables/PHASE_0_SUMMARY.md` with:

- Executive summary (one paragraph on state of the project)
- Architecture diagram (refined from `ARCHITECTURE_DRAFT.md`)
- Priority-ranked pain points with suggested remediation phases
- Personal learning goals checklist (Python concepts you encountered that you want to master)

**Deliverable:** Promote drafts to final form:

```bash
git mv docs/deliverables/ARCHITECTURE_DRAFT.md docs/deliverables/ARCHITECTURE.md
git mv docs/deliverables/CONFIG_SYSTEM_DRAFT.md docs/deliverables/CONFIG_SYSTEM.md
git commit -m "docs: promote Phase 0 drafts to final form"
```

Commit message body:

> Phase 0 codebase reconnaissance complete. All deliverables finalized. Ready to begin Phase 1 (tooling and code hygiene).

---

## Red Flags to Watch For

While reading, note these anti-patterns if you find them (candidates for Phase 1+ refactors):

- **Hardcoded paths:** Absolute paths like `/home/user/config.json`
- **Silent failures:** Errors swallowed without logging
- **String literals for config keys:** Instead of constants or enums
- **Monolithic functions:** Anything >100 lines doing multiple things
- **Duplicate code:** Same logic in multiple places for different providers
- **Magic numbers:** Unexplained values like `timeout = 30`
- **Global state:** Module-level mutable dicts or caches

---

## Phase 0 Learning Checklist

As you go, tag concepts you encounter that you want to learn more deeply:

|Concept|Why It Matters Here|Resource Suggestion|
|---|---|---|
|`pathlib`|File/path handling in config loading|Python docs + Cursor explainer|
|JSON parsing / serialization|Core to config system|MDN + Pydantic docs|
|Decorators|Likely used in routes/middleware|"Fluent Python" Ch. 7|
|Context managers|File handles, sessions, API connections|"Fluent Python" Ch. 14|
|Logging module|Observability, debugging|Python logging HOWTO|
|Virtual environments|Dependency isolation|pip & venv docs|
|Import resolution|How Python finds modules|Python import system docs|
|Dataclasses|Clean config/data structures|Python 3.10+ docs|
|Type annotations|Static analysis readiness|`typing` module docs|
|Exception handling patterns|Robust error messages|Python docs on exceptions|
|`requests` / `httpx`|API calls to providers|Library docs|
|Testing frameworks (`pytest`)|Validate changes|pytest docs + example tests|

Ask Cursor to explain any of these as you encounter them. Don't try to memorize them all upfront — let the project teach you.

---

## When You're Ready for Phase 1

You'll know Phase 0 is done when you can:

- ✅ Answer "What happens when RetroArch sends a game screenshot?" without looking at code
- ✅ Name each major module and what it owns
- ✅ Spot where a new provider would plug in
- ✅ Point to 3 UX problems worth solving

If you hit a wall on any of these, spend another day drilling deeper. It's better to spend 3 weeks here than to rush into refactoring and discover you missed a critical path.