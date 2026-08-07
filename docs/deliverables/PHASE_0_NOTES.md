# Phase 0 Notes

> Working document — first-pass readings, confusion points, and red flags.
> Documentation findings live in [DOCUMENTATION_MAP.md](DOCUMENTATION_MAP.md), not here.

## Module Purposes

One sentence each, filled in during the Day 1–2 reading pass. The 17 modules in
`src/vgtranslate3/`, to be read largest-first since they carry most of the logic.

| Module | Purpose (one sentence) |
| --- | --- |
| `serve.py` | |
| `util.py` | |
| `ocr_providers.py` | |
| `translation_providers.py` | |
| `ocr_tools.py` | |
| `local_ocr_providers.py` | |
| `local_translation_providers.py` | |
| `imaging.py` | |
| `opencv_engine.py` | |
| `bbox_extractor.py` | |
| `ocr_texter.py` | |
| `pyocr_util.py` | |
| `screen_translate.py` | |
| `server_client.py` | |
| `text_to_speech.py` | |
| `config.py` | |
| `__init__.py` | |

## Terms and Concepts Looked Up

Target 5–10. Cross-reference against the learning checklist in
[phase_0.md](../roadmap/phase_0.md).

## Clean vs. Messy

First impressions, before any refactoring bias sets in.

## Red Flags

Watching for: hardcoded paths, silent failures, string literals for config keys,
monolithic functions, duplicate provider logic, magic numbers, global state.

### Tooling and packaging (found while reading config files)

- **The runtime config is untrackable by design, but the build expects it.**
  `.gitignore` line 38 ignores `*config.json`, which matches
  `src/vgtranslate3/config.json` — the exact file every install instruction tells
  users to create. Meanwhile `pyproject.toml` lists `config.json` under
  `[tool.setuptools.package-data]`, so a build from a clean clone packages a file
  that cannot be in the clone. Worth confirming what a fresh `pip install .`
  actually produces.
- **Four overlapping quality tools, no stated authority.** The `dev` extra
  installs black, flake8, ruff, and mypy. Ruff's selected rules (`E`, `F`, `I`)
  duplicate flake8's job, and `E501` is explicitly deferred to black. Nothing
  documents which one a contributor is expected to satisfy.
- **`pytest-cov` is undeclared.** The coverage command in the Phase 0 plan needs
  it, and the `dev` extra does not list it, so `--cov` fails on a fresh sync.
  Already flagged as a Phase 1 task.

## Static Analysis

To be appended during Day 5–7: coverage estimate by module, untested critical
paths, ruff findings.
