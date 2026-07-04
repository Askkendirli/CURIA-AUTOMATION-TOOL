# Legal-Linguistic EU Case-Law AI Tool

A Windows-first local prototype for collecting public CURIA case-law pages, indexing them as a local archive, creating keyword case lists, generating concise legal-linguistic reports, and translating selected reports into EU official languages.

This repository is written as a research prototype. It is designed for legal-linguistic work where the relevant evidence is already public, but the manual process of finding, sorting, and preparing case notes is slow.

## What the prototype does

The tool separates the workflow into five stages:

1. **Collection**: CURIA pages are opened through a controlled browser session and saved as usable HTML and plain text.
2. **Local archive**: HTML, PDF, and text files are extracted without OCR and stored in a local SQLite database.
3. **Keyword case lists**: A local keyword search produces a human-readable Word list and a machine-readable JSON/CSV list.
4. **Report generation**: A local Ollama model creates short English legal-linguistic reports only for selected cases.
5. **Translation**: A local translation model translates selected reports into one or more EU official languages.

The default configuration is deliberately conservative so that testing does not time out on local hardware.

## Default safe model settings

The Windows-safe profile uses:

```yaml
analysis_model: "gpt-oss:20b"
translation_model: "translategemma:27b"
num_ctx: 8192
think: "low"
keep_alive: "0"
source_chars_per_case: 12000
max_cases_per_batch: 3
```

The smaller `gpt-oss:20b` model is used first because it is more suitable for proving that the archive, search, list, report, and translation steps work. `gpt-oss:120b` can then be tested with the included `config.120b.lowctx.example.yaml` profile.

## Installation on Windows

### 1. Install prerequisites

Install:

- Python 3.11 or newer
- Git for Windows
- Ollama for Windows

Open PowerShell in the repository folder.

If PowerShell blocks activation scripts, run this for the current PowerShell window only:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

### 2. Create the virtual environment

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
python -m playwright install chromium
python -m eu_case_ai setup --overwrite
```

### 3. Pull local models

```powershell
ollama pull gpt-oss:20b
ollama pull translategemma:27b
```

If the translation model is not available under that exact local name, edit `config.yaml` and replace `translation_model` with the local model tag shown by:

```powershell
ollama list
```

### 4. Check the installation

```powershell
python -m eu_case_ai doctor
python -m eu_case_ai model-test
```

The doctor command reports the number of indexed documents, the active models, `num_ctx`, and likely configuration problems.

## First working test

A small manual test is recommended before any bulk collection.

1. Save one known CURIA or EUR-Lex HTML document into:

```text
archive\raw\en\62024CC0416_en.html
```

2. Ingest it:

```powershell
python -m eu_case_ai ingest archive\raw\en\62024CC0416_en.html --lang en --celex 62024CC0416
```

3. Create a keyword case list without using the LLM:

```powershell
python -m eu_case_ai search --keywords "language version; legal certainty" --match-mode any --context ""
```

4. Generate a report only after the list exists:

```powershell
python -m eu_case_ai report --list archive\case_lists\NAME_OF_LIST.json --context "translation error recovery"
```

5. Translate the report if required:

```powershell
python -m eu_case_ai translate --report reports\en\NAME_OF_REPORT.md --targets ro,de,fr
```

## Guided interactive mode

```powershell
python -m eu_case_ai workflow
```

The guided mode starts by asking whether the work should proceed by:

1. specifying local criteria and keywords,
2. choosing a recent keyword case list, or
3. importing a specific keyword list from another folder.

The bulk collection option is also available from the same menu.

## Bulk CURIA collection

The collector is integrated into the main project and is Windows-first. It is based on the earlier standalone Mac collector, but it now records collection coverage, saves manifests into the main archive, and can ingest collected HTML automatically.

Small test:

```powershell
python -m eu_case_ai collect --years 2024 --courts C --langs en --max-case-number 30 --max-saved 3 --headed --manual
```

Larger collection:

```powershell
python -m eu_case_ai collect --years 2024-2025 --courts C,T --langs en --max-case-number 900 --headed --manual
```

The collection method probes case numbers inside the specified year and court prefix. Missing cases are normal because not every generated identifier corresponds to a published document. The manifest records saved documents and missing/blocked pages.

## Keyword list workflow

A keyword search produces three files:

```text
archive\case_lists\keyword_list_*.docx   human-readable Word list
archive\case_lists\keyword_list_*.json   machine-readable list
archive\case_lists\keyword_list_*.csv    spreadsheet-friendly list
```

If no context is supplied, the workflow stops after the list is generated. This avoids unnecessary model calls. If a short context is supplied, report generation becomes available.

## Report workflow

Reports are generated in English first. They are intentionally concise and structured for legal-linguistic reading:

- abstract,
- facts and linguistic problem,
- procedural and legal context,
- language variation or terminology issue,
- method of interpretation,
- outcome and significance,
- relevance to the supplied context.

The tool processes reports case by case, not as one large batch. This reduces timeout risk.

## Translation workflow

Reports may be translated into any official EU language code:

```text
bg, es, cs, da, de, et, el, fr, ga, hr, it, lv, lt, hu, mt, nl, pl, pt, ro, sk, sl, fi, sv
```

English reports are already the source reports and are not retranslated to English.

## Troubleshooting summary

Run:

```powershell
python -m eu_case_ai doctor
python -m eu_case_ai model-test
```

If a model times out or fails to load, lower the following values in `config.yaml`:

```yaml
llm:
  num_ctx: 4096
  think: "low"
reports:
  source_chars_per_case: 8000
  max_cases_per_batch: 1
search:
  rerank_limit: 5
```

If the 120B model fails before generation begins, the issue is usually model loading memory rather than report logic. The 20B profile should be used to validate the workflow first.

Detailed troubleshooting is in [`docs/TROUBLESHOOTING.md`](docs/TROUBLESHOOTING.md).

## Repository structure

```text
eu_case_ai/                 Python package
scripts/                    Windows helper scripts
docs/                       methodology, troubleshooting, technical appendix
presentation/               presentation material
examples/                   imported list example
seeds/                      small known-case seed file
legacy_mac_collector/       original standalone Mac collector retained for reference
```

## Research disclaimer

The tool is an aid for legal-linguistic research. It does not replace legal analysis, source verification, or revision by a qualified researcher. Generated reports must be checked against the underlying case documents.
