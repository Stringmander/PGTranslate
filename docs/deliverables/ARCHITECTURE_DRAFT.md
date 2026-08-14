# Architecture (Draft)

> Working document — Phase 0 WIP. Will be promoted to ARCHITECTURE.md upon phase completion.

> **How this file was prepared.** The lifecycle diagram, data-flow table, module
> dependencies, and findings below were derived by tracing the code and are
> verified against it — line references point at the relevant source. The
> **Responsibility** entries and the one-sentence summary are deliberately left
> empty: those are the Phase 0 success criteria, and filling them in is the
> point of the reading pass.

**One-Sentence Summary:** PGTranslate is a ...

## Request Lifecycle

```text
RetroArch
   │  POST /?source_lang=jpn&target_lang=en&mode=fast&output=image,sound
   │  body: {"image": "<base64>"}
   ▼
APIHandler.do_POST                                        serve.py:84
   │  json.loads(body)                          ← no error handling
   ▼
APIHandler._process_request                               serve.py:119
   │  parse query → {"image": "bmp"|"png", "sound": "wav"}
   │  util.load_image(b64) → PIL RGBA → .convert("RGB")
   │
   ├─ api_key_type == "free"       → {"error": "not implemented"}
   ├─ api_key_type == "ztranslate" → screen_translate.CallService
   │                                   └→ server_client → ztranslate.net
   ▼
APIHandler._handle_modern_pipeline               serve.py:213 (275 lines)
   │
   ├─ 1  resolve providers from config, with legacy api_key_type overrides
   │
   ├─ 2  ocr_providers.get_ocr_provider(name)        ocr_providers.py:576
   │        └─ .recognize(image_data, source_lang) → (data, raw_output)
   │           retried ×3 on TypeError/KeyError/IndexError only
   │
   ├─ 3  normalise blocks: bbox → boundingBox.vertices, text → source_text
   │        skipped entirely when the provider is "tesseract"   serve.py:284
   │
   ├─ 4  translation_providers.get_translation_provider(name)
   │        └─ .translate(blocks, target_lang, source_lang)
   │
   ├─ 5  write translation in both shapes: translation_str (legacy string)
   │        and translation{target_lang} (dict)                 serve.py:322
   │
   ├─ 6  TTS, if config.tts_enabled and "sound" requested
   │        └─ TextToSpeech.text_to_speech_api → wav → _fix_wav_size → b64
   │
   ├─ 7  imaging.ImageModder.write(image, data, target_lang)   imaging.py:230
   │        └─ util.image_to_string_format → base64 BMP/PNG
   │
   └─ 8  Web UI broadcast, via asyncio.run() per request        serve.py:483
   ▼
{"image": "<b64>", "sound": "<b64>", "error": "<str>"}
                                   ↑ note: no "blocks" key    serve.py:451
```

## Module Breakdown

Seventeen modules in `src/vgtranslate3/`, plus the Web UI package. The table
below is the index; the responsibility and notes for each module follow it.
Dependencies and notes are pre-verified — write the responsibility as you read.

| Module | Lines | Internal deps |
| --- | ---: | --- |
| `util.py` | 817 | none |
| `ocr_providers.py` | 589 | `config`, `bbox_extractor`, `util` |
| `serve.py` | 586 | `config`, `imaging`, `ocr_providers`, `screen_translate`, `translation_providers`, `text_to_speech`, `util`, `bbox_extractor`, `webui.server` |
| `translation_providers.py` | 582 | `config`, `util` |
| `ocr_tools.py` | 503 | `util`, `pyocr_util` |
| `imaging.py` | 419 | `util` |
| `config.py` | 343 | `imaging` |
| `text_to_speech.py` | 279 | `config` |
| `local_ocr_providers.py` | 270 | `config`, `bbox_extractor`, `util` |
| `opencv_engine.py` | 218 | none |
| `local_translation_providers.py` | 191 | `config` |
| `pyocr_util.py` | 124 | none |
| `server_client.py` | 82 | `config` |
| `bbox_extractor.py` | 72 | none |
| `screen_translate.py` | 67 | `config`, `imaging`, `server_client` |
| `__init__.py` | 38 | none |
| `ocr_texter.py` | 6 | none |
| `webui/server.py` | ~190 | none |

## Per-Module Assessment

### `util.py`

**Responsibility.** Owns no single concern — it is the project's dependency-free
bottom layer, bundling four unrelated toolkits behind one namespace: base64↔PIL
image codec, colour reduction and pixel preprocessing for OCR, bounding-box
geometry, and the ISO 639-1→639-3 language map.

**Notes.** Largest module, ~35 free functions. No internal imports, so it is the
leaf of the graph — everything depends on it, it depends on nothing.
`lang_2_to_3` (:15) maps `sk` to `skk`, which is Sok; Slovak is `slk`.

### `ocr_providers.py`

**Responsibility.** _Not yet written._

**Notes.** Five providers behind `get_ocr_provider` (:576). Imports
`bbox_extractor` unguarded at :20, which makes OpenCV a hard requirement — see
finding 2.

### `serve.py`

**Responsibility.** The server, and the only module that knows the pipeline as a
whole: it terminates the RetroArch HTTP request, resolves which providers to use
from config, then drives OCR, block normalisation, translation, TTS, rendering,
and the Web UI broadcast in that order. Structurally it is the mirror of
`util.py` — the root of the dependency graph rather than the leaf, importing
nine of the eighteen modules and imported by none. That position is why every
concern that belongs to no one else has settled here: the legacy ztranslate
branch, the dead desktop-GUI branch, and a five-line WAV header patcher.

**Notes.** Entry point, via the `vgtranslate3` console script.
`_handle_modern_pipeline` (:213) is 275 lines and takes twelve parameters, four
of which — `body`, `mode`, `request_output`, `alpha` — are never referenced in
the body; a fifth, `pixel_format`, appears only in the unreachable BGR branch.
Also holds a second, threaded server path (`start_api_server`, :508) for a GUI
window object absent from this codebase — inherited from upstream, unreachable,
see finding 11.

### `translation_providers.py`

**Responsibility.** _Not yet written._

**Notes.** Six providers behind `get_translation_provider` (:568). OpenAI
(:151), DeepSeek (:290), and Groq (:394) are near-identical — see finding 7.

### `ocr_tools.py`

**Responsibility.** _Not yet written._

**Notes.** Tesseract helpers, forked per platform:
`tess_helper_windows`/`_linux`/`_server` plus `_data_` variants of each. Has its
own `main()` at :494.

### `imaging.py`

**Responsibility.** _Not yet written._

**Notes.** Text rendering onto the output image (`ImageModder.write`, :230),
plus `ImageSaver`/`ImageIterator` for an on-disk image history used only by the
GUI path.

### `config.py`

**Responsibility.** _Not yet written._

**Notes.** Module-level globals as the config store. `load_init()` (:97) is 229
lines. Calls `imaging.load_font()` at :321, so loading config has a font side
effect.

### `text_to_speech.py`

**Responsibility.** _Not yet written._

**Notes.** Google/Yandex/OpenAI TTS behind `text_to_speech_api` (:24). Has its
own `main()` at :272.

### `local_ocr_providers.py`

**Responsibility.** _Not yet written._

**Notes.** Ollama and vLLM OCR. **Not imported by the server** — only by
`tests/test_providers.py`. See finding 1.

### `opencv_engine.py`

**Responsibility.** _Not yet written._

**Notes.** Standalone script (imports `argparse`). Unreferenced anywhere in
`src/`.

### `local_translation_providers.py`

**Responsibility.** _Not yet written._

**Notes.** Ollama and vLLM translation. **Not imported by the server** — only by
tests. See finding 1.

### `pyocr_util.py`

**Responsibility.** _Not yet written._

**Notes.** Wraps the `pyocr` libtesseract backend; imported by `ocr_tools`
inside a `try`.

### `server_client.py`

**Responsibility.** _Not yet written._

**Notes.** HTTP client for the legacy ztranslate.net service.

### `bbox_extractor.py`

**Responsibility.** _Not yet written._

**Notes.** OpenCV contour detection for bounding boxes; imports `cv2` and
`numpy` at module scope.

### `screen_translate.py`

**Responsibility.** _Not yet written._

**Notes.** Legacy ztranslate request path, reached only via `api_key_type ==
"ztranslate"`.

### `__init__.py`

**Responsibility.** _Not yet written._

**Notes.** Exposes `load_default_config()` (:30).

### `ocr_texter.py`

**Responsibility.** _Not yet written._

**Notes.** A single classmethod stub. Unreferenced anywhere.

### `webui/server.py`

**Responsibility.** _Not yet written._

**Notes.** WebSocket broadcast server and in-memory history, imported by
`serve.py` inside a `try`.

## Data Flow Transformation Points

| Stage | Input | Output | Transform |
| --- | --- | --- | --- |
| HTTP receipt | Raw request body + query string | `dict` | `json.loads`; query split on `&` then `=` (serve.py:89) |
| Decode | base64 string | `PIL.Image` RGBA, then RGB | `util.load_image` (:101) converts to RGBA; caller immediately converts to RGB |
| OCR | The **original** base64 string, not the decoded image | `({"blocks": [...]}, raw_response)` | Provider `.recognize`; each provider re-encodes as it sees fit |
| Normalise | Blocks with `bbox` / `text` | Blocks with `boundingBox.vertices` / `source_text` | serve.py:284–306; bypassed for Tesseract |
| Translate | `list[block]`, target, source | `{"blocks": [...]}` with translations | Provider `.translate` |
| Dual-format | `block["translation"]` as `str` or `dict` | `translation_str` **and** `translation{lang}`, plus `target_lang` | serve.py:322–352; also rewrites `bounding_box` from x1/y1/x2/y2 to x/y/w/h |
| TTS | Blocks sorted by (y, x), joined with spaces | WAV bytes → base64 ascii | `TextToSpeech.text_to_speech_api`, then `_fix_wav_size` patches RIFF length fields |
| Render | RGB image + blocks | `PIL.Image` with translated text drawn in | `imaging.ImageModder.write` (:230) |
| Encode | `PIL.Image` | base64 BMP or PNG | `util.image_to_string_format` (:118); falls back to BMP inside a bare `except` |
| Response | `dict` | JSON bytes | `json.dumps(ensure_ascii=False)`, UTF-8 |

## Notes from Reading

Findings below were verified against the code; each names the evidence.

**1. The local-model providers are unreachable from the server.** `get_ocr_provider`
accepts only google, yandex, openai, gemini, tesseract; `get_translation_provider`
only google, yandex, openai, deepseek, groq, gemini. Neither `ollama` nor `vllm`
appears in either table, and `local_ocr_providers` / `local_translation_providers`
are imported by `tests/test_providers.py` and nothing else. Confirmed by running
the dispatch functions in the project venv:

```text
OCR          ollama   -> ValueError: Unknown OCR provider: ollama
TRANSLATION  ollama   -> ValueError: Unknown translation provider: ollama
OCR          vllm     -> ValueError: Unknown OCR provider: vllm
```

Three of the ten shipped example configs — `config_ollama_local.json`,
`config_vllm_local.json`, `config_hybrid_ollama_groq.json` — select these
providers, so all three fail. `LOCAL_MODELS_GUIDE.md` documents the feature at
length, including instructions to edit `local_ocr_providers.py`. The 461 lines of
provider code exist and are unit-tested; only the wiring into `get_*_provider` is
missing.

The wiring was never there. A pickaxe search over every commit on every branch
(`git log --all -S ollama -- src/vgtranslate3/ocr_providers.py
src/vgtranslate3/translation_providers.py`) returns nothing for `ollama` and
nothing for `vllm`, while the same search for `gemini` correctly finds
`2e6d08e`. The commit that introduced the feature, `7aae9ea` ("Add full
on-device translation support (Ollama & vLLM)", 2026-03-07), touched seven
files — the two provider modules, `config.py`, three example configs, and
`LOCAL_MODELS_GUIDE.md` — and neither dispatch module is among them. The
providers shipped unwired on day one; this is not a regression from the fork or
the 3.14 migration.

The failure also surfaces badly: `get_ocr_provider` is called at serve.py:252,
outside the retry `try` block, and `do_POST` has no exception handling, so the
`ValueError` escapes the handler rather than becoming the `{"error": ...}`
response the code builds elsewhere.

**2. OpenCV is a hard dependency despite being packaged as optional.**
`serve.py:32` wraps `bbox_extractor` in `try/except ImportError` with fallback
stubs — but `ocr_providers.py:20` imports it unguarded, and `serve.py:21` imports
`ocr_providers` first. So the fallback can never trigger. Verified: importing
`ocr_providers` without `cv2` raises `ModuleNotFoundError` at
`bbox_extractor.py:1`. `pyproject.toml` lists `opencv-python-headless` under the
optional `local-ocr` extra, so a base `pip install` yields a server that cannot
start. `requirements.txt` installs it unconditionally, which is why the
documented path masks the problem.

**3. The server is single-threaded.** `main()` uses `http.server.HTTPServer`
(:566), not `ThreadingHTTPServer`, so a full OCR-plus-translation round trip
blocks the next request.

**4. Error handling protects the wrong failure mode.** The OCR retry
(serve.py:256) catches `TypeError`, `KeyError`, `IndexError` — response-parsing
errors — but not network errors or timeouts, which is what actually fails when
calling a remote provider. Nothing wraps `do_POST`, so any uncaught exception
kills the response instead of returning the error JSON.

**5. Config is unvalidated module-level global state.** `load_init()` is 229
lines: roughly 60 `global` declarations followed by ~70 repetitions of
`if "key" in config_file: key = config_file["key"]`. Unknown keys are silently
ignored, missing keys silently keep defaults, and no value is type-checked or
range-checked. A typo like `openai_api_ke` is discarded without a word. Any
failure to read the file — including "not found" — prints the same "Invalid
config file specification".

**6. One config key is silently dropped.** config.py:276:

```python
if "yandex_llm_api_key" in config_file:
    config_file["yandex_llm_api_key"]      # expression, not assignment
```

The value is read and thrown away. Correspondingly, `global yandex_llm_api_key`
is missing from the declarations, while `global yandex_ocr_key` appears twice
(:109 and :153) — consistent with a copy-paste slip where the second should have
been the LLM key.

**7. The OpenAI-compatible providers are triplicated.** `OpenAITranslationProvider`
(:151), `DeepSeekTranslationProvider` (:290), and `GroqTranslationProvider`
(:394) are near-identical, differing mainly in which config keys they read —
despite config.py:39 and :45 marking the DeepSeek and Groq settings deprecated in
favour of `openai_*`. `ocr_tools.py` has the same shape, forked by platform.

**8. The response contract contradicts the benchmark guide.** serve.py:451
deliberately omits `blocks`, returning only image, sound, and error.
`tests/benchmark/README.md` instructs readers to parse
`data['blocks'][0]['source_text']`, which cannot work against this server —
independent of the missing fixture files already recorded in the documentation
map.

**9. Dead and vestigial code.** `pixel_format` is hardcoded to `"RGB"`
(serve.py:144), making both `if pixel_format == "BGR"` branches unreachable.
`g_debug_mode` is declared `global` in `main()` but never defined at module scope
and never read. `config.write_init()` (:328) has no callers, writes to a
cwd-relative `./config.json` rather than `CFG_PATH`, and persists only 11 of the
~60 settings. `ocr_texter.py` and `opencv_engine.py` are referenced nowhere.

**10. Query parsing can crash on valid input.** serve.py:89 builds a dict via
`qc.split("=")` per parameter, which raises `ValueError` if any value contains a
second `=`, and never URL-decodes.

**11. The GUI seam is unreachable, and it is inherited.** `serve.py` carries a
second server entry point built for a desktop window that is not in this
repository: `window_obj` (:72), `start_api_server(window_object)` (:508),
`kill_api_server` (:517), and `start_api_server2` (:524). Nothing calls any of
them — `start_api_server`, `kill_api_server`, and `load_image_object` appear
nowhere in the tree outside `serve.py` itself. Since `window_obj` is only ever
reassigned by `start_api_server`, it is permanently `None`, so the
`if window_obj:` branch at :421–424 is dead. That branch holds the only
`ImageIterator` reference in `serve.py`, which means the server never reads the
on-disk image history during normal operation.

All of it predates the fork: the same code sits at lines 68 and 415 of
`c44b55d` ("Чистка 2", 2026-03-08), the last upstream commit before this fork,
unchanged since apart from blank lines inserted between the definitions.
Comparing `serve.py` at that commit against HEAD as parsed syntax
trees — which discards formatting entirely — the only real changes are dropping
`from __future__ import annotations`, removing thirteen imports that each
occurred exactly once (on their own import line), rewriting `Dict` and
`List[str]` as `dict` and `list[str]` in three signatures, and deleting a dead
`error_string = ""` at the top of `_process_request` that was assigned and never
read. The remaining ~230 lines of the raw diff are Black reformatting. The
module defines the same fourteen functions and classes before and after, and
`ruff --select F821,F401` reports no undefined names and no unused imports.

## Unknowns & Questions

- [ ] Does RetroArch send `output=image,sound` or something narrower, and does it depend on `blocks` ever?
- [ ] What is `mode` (`fast` vs `normal`) meant to change? It only affects the ztranslate path today.
- [x] ~~Was the local-model wiring ever present, or did the providers land without it?~~ The providers landed without it. `7aae9ea` (2026-03-07) added both modules, their config keys, three example configs, and the guide, but never touched either dispatch module — and no commit on any branch has ever referenced `ollama` or `vllm` from one. Finding 1.
- [x] ~~Is the threaded `start_api_server` path reachable at all, or purely vestigial from the GUI original?~~ Purely vestigial, and inherited rather than introduced by the fork. No callers anywhere in the tree; `window_obj` is permanently `None`; already present at the pre-fork commit `c44b55d`. Finding 11.
- [ ] Which provider does `local_server_api_key_type` override, and why does it coexist with `ocr_provider`?
- [ ] Does `ImageIterator`'s on-disk history write anything during normal server operation? Partly answered: `serve.py`'s only `ImageIterator` reference sits inside the dead GUI branch (finding 11), so the request path never touches it. Still to check whether `imaging.ImageSaver` writes from anywhere live.
- [ ] What consumes `translation_str` — is any real client still reading the legacy string shape?
