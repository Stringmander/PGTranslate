# Documentation Map

Inventory of existing project documentation, assessed during Phase 0 reconnaissance.
Line references were verified against the tree as of the Phase 0 reading pass.

## Inventory

| Doc | Purpose | Audience | Completeness |
| --- | --- | --- | --- |
| [README.md](../../README.md) | Project overview, quickstart, provider table, Docker, RetroArch | New users and contributors | Complete but over-scoped |
| [INSTALL.md](../../INSTALL.md) | Setup, system requirements, import-error troubleshooting | New users | Partial |
| [TESSERACT_GUIDE.md](../../TESSERACT_GUIDE.md) | Installing, configuring, and tuning local Tesseract OCR | Users configuring Tesseract for OCR | Strongest of the guides, with defects |
| [LOCAL_MODELS_GUIDE.md](../../LOCAL_MODELS_GUIDE.md) | Running OCR and translation against local Ollama or vLLM | Users configuring local models | Comprehensive but redundant |
| [tests/benchmark/README.md](../../tests/benchmark/README.md) | Manual quality-testing methodology and scoring rubric | New users and contributors | Aspirational |

## Per-Document Assessment

### README.md

**Covers**

- What the project is, and how it relates to upstream VGTranslate and ztranslate
- A clone-to-running installation sequence
- Provider selection, as a table mapping each provider to its example config
- Settings shared by all OpenAI-compatible providers, presented as the common case
- Docker build and run, including overriding the port
- Invocation of the three test entry points
- RetroArch setup and the server's network binding behaviour
- Licence, credits, and the author's framing of the project as a personal PoC

**Assessment.** Complete as an overview, but over-scoped. It restates install,
configuration, and testing material that belongs in the dedicated documents,
which means those topics now have two homes that can disagree — and on the run
command and clone URL, they already do.

### INSTALL.md

**Covers**

- The same clone-to-running sequence as the README, plus Windows venv activation
- Optional Tesseract system packages for Debian, macOS, and Windows
- Minimum Python version and RAM
- Docker build and run
- Fixes for three missing-module import errors
- Pointers to the two guides and the example config directory

**Assessment.** Partial. Troubleshooting addresses only missing Python packages
and stops well short of the failures users actually hit. The Russian translation
covers the Quick Start code comments and then stops, so a reader arriving in
Russian silently loses translation from "Optional: Local OCR" onward.

### TESSERACT_GUIDE.md

**Covers**

- Tesseract installation for Linux, macOS, and Windows, including language packs
- Choosing a starting config, and what the OCR processor fields mean
- A PSM mode reference table, with a recommendation for game UI
- The three preprocessing actions and their options
- Worked pipelines for Japanese visual novels, noisy Russian, English subtitles
- Tuning advice for threshold, contrast, bbox fallback, and image size
- Troubleshooting organised by symptom
- A Tesseract-versus-cloud comparison and a closing recommendation

**Assessment.** The strongest of the guides. It is the only document that covers
all three platforms evenly, and organising troubleshooting by symptom rather than
by error string is the right instinct. Two defects undercut it: it points at a
`config_tesseract_rus.json` that does not exist, and its pipeline examples are
labelled as JSON while containing `//` comments, so the samples it invites the
reader to copy cannot be parsed.

### LOCAL_MODELS_GUIDE.md

**Covers**

- Why run models locally, with an architecture diagram
- Ollama installation for Linux/macOS and Windows, and which models to pull
- vLLM installation and server invocation
- Hardware requirements by model size
- Configs for Ollama-only, vLLM-only, and hybrid local-OCR-with-cloud-translation
- Model recommendations
- Measured processing times per configuration
- Troubleshooting for connection failures and poor output quality
- Privacy, cost, and speed comparisons against cloud providers
- Advanced topics: custom prompts, multi-GPU serving, quantisation
- Exercising the pipeline without API keys

**Assessment.** Comprehensive but redundant. The same local-versus-cloud tradeoff
is argued in four separate sections, which makes the document longer than its
content warrants and gives the numbers room to disagree with each other. Platform
coverage is also asymmetric: Ollama gets both Linux/macOS and Windows
instructions, while vLLM is documented for Linux with CUDA only, with no mention
that Windows users need WSL2.

### tests/benchmark/README.md

**Covers**

- Four recommended test-image categories, with the OCR challenge each represents
- 1–5 scoring rubrics for OCR accuracy, translation quality, and bbox alignment
- A manual end-to-end procedure: prepare image, start server, request, parse, inspect
- A score sheet and a test-log template
- Star-rated provider comparisons across cloud, local, and Tesseract pipelines
- A per-provider checklist spanning languages, font sizes, backgrounds, and TTS
- Invocation of a semi-automated benchmark runner
- Expected accuracy baselines per provider
- What to include when reporting a quality issue

**Assessment.** Aspirational. The scoring rubric is genuinely well designed —
concrete percentage bands rather than vague adjectives, and separate axes for OCR
and translation so a failure can be attributed. But every artifact it tells the
reader to run or open is absent: the runner script, both test images, and the
example config it loads. Its parsing snippet also does not run. The result is a
methodology document describing a process no reader can currently follow, which
in turn makes its stated performance baselines unreproducible.

## Gaps

- **No RetroArch protocol documentation.** The integration is the project's entire reason for existing, and it gets three bullet points in [README.md](../../README.md). Nothing documents the endpoint, query parameters, request body, or response schema. The `output=image` and `source_lang`/`target_lang` parameters appear only incidentally, inside a `curl` example in the benchmark guide.
- **No configuration reference.** Ten `config_*.json` examples ship in `src/vgtranslate3/config_example/`, and no document explains what any individual field means. [README.md](../../README.md) maps provider to filename; [TESSERACT_GUIDE.md](../../TESSERACT_GUIDE.md) explains the three preprocessing actions. Everything else — `min_pixels`, `use_bbox_fallback`, `local_server_host`, TTS settings — is discoverable only by reading source.
- **No contributor or development documentation.** `pyproject.toml` configures pytest, ruff, mypy, and black, and none of that is written down anywhere. There is no CONTRIBUTING file, no description of the `uv` workflow, and no statement of which linter is authoritative (the `dev` extra installs both flake8 and ruff).
- **The Web UI is undocumented.** `src/vgtranslate3/webui/` exists and `pyproject.toml` declares a `webui` extra, but no document mentions the feature, how to enable it, or what it does.
- **Bilingual coverage is inconsistent.** [TESSERACT_GUIDE.md](../../TESSERACT_GUIDE.md) and [LOCAL_MODELS_GUIDE.md](../../LOCAL_MODELS_GUIDE.md) translate headings and prose. [INSTALL.md](../../INSTALL.md) translates only the comments inside the Quick Start code block — from "Optional: Local OCR" onward it is English-only. A reader arriving in Russian gets a guide that silently stops being bilingual halfway through.
- **No troubleshooting for the server itself.** [INSTALL.md](../../INSTALL.md) troubleshoots three missing Python modules. Nothing addresses the failures users will actually hit: port already in use, RetroArch cannot reach the host, malformed config, expired or rejected API key.
- **No baseline test assets.** `tests/benchmark/` contains a README and nothing else. Readers are told to score OCR accuracy as a percentage against images they must supply themselves, which makes the stated baselines unfalsifiable.
- **Existing documentation sits outside `docs/`.** Four guides live in the repository root and one under `tests/benchmark/`, so a reader looking for a specific topic has no single place to start. Recorded as current state rather than as an open question: consolidating them under `docs/`, leaving only [README.md](../../README.md) in the root, happens once reconnaissance is complete, and [roadmap.md](../roadmap/roadmap.md) carries it in Phase 1. Phase 0 output under `docs/roadmap/` and `docs/deliverables/` is already in its final location and is not part of this.

## Stale, Duplicated, and Contradictory

### Broken references

Every path below is referenced by a document and absent from the repository:

| Referencing doc | Reference | Problem |
| --- | --- | --- |
| [README.md](../../README.md) | `tests/benchmark/benchmark.py` | Script does not exist |
| [tests/benchmark/README.md](../../tests/benchmark/README.md) | `tests/benchmark/run_benchmark.py` | Script does not exist — and is a second name for the same missing script |
| [tests/benchmark/README.md](../../tests/benchmark/README.md) | `test_image_jpn.png`, `test_jpn_vn_001.png` | Neither image exists; the two names refer to the same intended fixture |
| [tests/benchmark/README.md](../../tests/benchmark/README.md) | `config_example/config_openai.json` | No such example config; the OpenAI-compatible example is `config_routerai.json` |
| [TESSERACT_GUIDE.md](../../TESSERACT_GUIDE.md) | `config_example/config_tesseract_rus.json` | No such example config |

Separately, [TESSERACT_GUIDE.md](../../TESSERACT_GUIDE.md) and [tests/benchmark/README.md](../../tests/benchmark/README.md) both write the config path as `config_example/...`, while [README.md](../../README.md) and [INSTALL.md](../../INSTALL.md) correctly write `src/vgtranslate3/config_example/...`. The short form fails from the repository root.

### Contradictions with the actual project

Distinguish rot from pending work. Items here contradict the repository as it
stands, but some are known future refactors that Phase 0 deliberately does not
touch — Phase 0 is analysis only. Those are marked as tracked.

- **Documented run command is not the packaged one.** All docs say `python -m src.vgtranslate3.serve`. `pyproject.toml` declares the console script `vgtranslate3 = "vgtranslate3.serve:main"` with `package-dir = {"" = "src"}`. The documented form works only from the repository root and treats `src` as a package, which the packaging configuration says it is not.
- **Documented install contradicts the dependency model.** All docs say `pip install -r requirements.txt`. That file flattens the `local-ocr` and `webui` extras into a single required list, so a cloud-only user is made to install OpenCV, numpy, and websockets. `pyproject.toml` models these as optional extras. The two dependency sources will drift.
- **Project name predates the fork rename — tracked, not rot.** Every document says VGTranslate3, as do `pyproject.toml`'s package name and its `vgtranslate3` console script. This is pending refactor work, not documentation decay, and renaming is out of scope for an analysis-only phase.

  Worth noting, though: no phase actually *schedules* the rename. [roadmap.md](../roadmap/roadmap.md) assumes the new name already exists by Phase 2 (`pgtranslate init`) and Phase 5 (`pip install pgtranslate`), without a step that performs the change. It needs an owner — most naturally Phase 1, alongside the other hygiene work, so that Phase 2 can build the CLI under its final name rather than renaming an entry point it just shipped.

  The one part that affects a reader today is separate from the rename: [README.md](../../README.md) instructs readers to clone `Elveman/VGTranslate3`, so following the quickstart verbatim checks out upstream rather than this fork.
- **Sample code does not run.** [tests/benchmark/README.md](../../tests/benchmark/README.md) contains `Image.open(Image.open(img_data))` — `Image.open` applied twice, to raw bytes rather than a file object. It needs to wrap the decoded bytes in `io.BytesIO`.
- **JSON samples are not valid JSON.** The pipeline documentation in [TESSERACT_GUIDE.md](../../TESSERACT_GUIDE.md) annotates ` ```json ` blocks with `//` comments. Copy-pasted verbatim — which is exactly what the surrounding prose invites — the parser rejects them.

### Duplication

- **The quickstart exists twice**, in [README.md](../../README.md) and [INSTALL.md](../../INSTALL.md), differing only in the Windows activation comment and a slightly longer configure step. Two copies of an install sequence that is already wrong about the run command means fixing it requires remembering both.
- **The cost/privacy/speed tradeoff is argued repeatedly.** [LOCAL_MODELS_GUIDE.md](../../LOCAL_MODELS_GUIDE.md) makes essentially the same local-versus-cloud case in "Performance Comparison", "Privacy and Security", "Cost Comparison", and its closing recommendation. [TESSERACT_GUIDE.md](../../TESSERACT_GUIDE.md) and [tests/benchmark/README.md](../../tests/benchmark/README.md) each make it a third and fourth time, with numbers that are not consistent between them.

### Presentation

- [INSTALL.md](../../INSTALL.md) closes with a "More Information" section listing sibling docs as bare filenames rather than links, as does its Tesseract pointer. [README.md](../../README.md) links its documentation section properly and is the model to follow — though it omits [tests/benchmark/README.md](../../tests/benchmark/README.md) entirely.
- [TESSERACT_GUIDE.md](../../TESSERACT_GUIDE.md) opens with "VGTranslate3 **now** supports Tesseract OCR" — changelog voice in reference documentation, with no date to anchor "now".

## Recommendations

Scoped to the existing documentation. The migration of these files into `docs/`
is already planned for after reconnaissance and is assumed below rather than
proposed.

1. Write documentation in English only and add an internationalization workflow, rather than maintaining inline bilingual prose. The current approach has already decayed unevenly across files.
2. When the guides move under `docs/`, have [README.md](../../README.md) link out to them rather than restate their content. The move only pays off if the root README stops being a second copy.
3. Make the quickstart exist in exactly one place.
4. Treat the benchmark guide as unshippable until the fixture images and runner script exist. Until then it describes a testing process no reader can follow.
