# OCR-LLM

A Python pipeline that extracts structured data from scanned PDF forms using a Visual LLM. Forms are identified by QR codes embedded in each page, by the subfolder they are placed in, or by an LLM classifier when a single PDF contains pages from multiple form types. Results are written to JSON files and appended to Excel files. A browser-based review app lets you inspect and correct extracted data.

---

## Installation

### 1. Install Python

Download and install Python 3.12 or later from [python.org](https://www.python.org/downloads/). During installation, check **"Add Python to PATH"**.

Verify the installation:

```bash
python --version
```

### 2. Install the package

From the project root:

```bash
pip install -e .
```

This installs `dhocr` package in editable mode and registers the `dhocr-batch` and `dhocr-review` commands.

---

## Input Folder Structure

The pipeline scans up to three input folders:

| Folder | Description |
|---|---|
| `input/Forms/` | PDFs of a single form type. The form type is determined by the immediate subfolder name. |
| `input/Forms_with_QR/` | PDFs with multiple form types. The form type of each page is determind by the QR code on the page. |
| `input/Forms_mixed/` | PDFs with multiple form types and without QR code (Mixed-form PDF). The form type of each page is determined by the LLM classifier using the prompt in the form_selector.yaml config file described below|

***WARNING***: Mixed-form PDFs may require each page to be passed to the LLM twice and double the processing time and system resources. 
Only use it when the benefits are justified. 

All folders support arbitrary subfolders. The subfolder structure is mirrored in the output.

### Single-form PDFs (`input/Forms/`)

Place PDFs under a subfolder named after the form ID:

```
input/Forms/
  FHS-Q1-TC-v1/
    batch1/
      scan_001.pdf
    scan_002.pdf
```

- The form ID is taken from the **first level** subfolder under `input/Forms/` (e.g. `FHS-Q1-TC-v1`).
- Pages are grouped sequentially using `num_pages` from the form config.
- Deeper subfolders (e.g. `batch1/`) are preserved in the output path.

### QR PDFs (`input/Forms_with_QR/`)

Place PDFs anywhere under this folder. Each page must contain a QR code encoding:

```json
{"form_id": "F001", "page": 1}
```

The pipeline detects form boundaries automatically when the `form_id` changes or page numbering resets.

### Mixed-form PDFs (`input/Forms_mixed/`)

Place PDFs anywhere under this folder. Pages are sent to the LLM with the classifier prompt defined in `config/form_selector.yaml` for classification of the form ID. The LLM returns:

```json
{"form_id": "F001"}
```

The LLM is called only for the first page of each form instance. Subsequent pages are assigned sequential page numbers automatically using `num_pages` from the form config (`num_pages` is required for all forms used in mixed mode). Pages where the classifier fails are skipped with a warning.

Before using this mode, edit `config/form_selector.yaml` to describe your form types and their identifying conditions (see [Form selector config](#form-selector-config--configform_selectoryaml) below).

---

## Configuration

### Global config — `config/global_config.json`

```jsonc
{
  "forms_input_folder":      "input/Forms",           // Single-form PDF input folder
  "forms_output_folder":     "output/Forms",           // Single-form PDF output folder
  "forms_qr_input_folder":   "input/Forms_with_QR",   // QR PDF input folder
  "forms_qr_output_folder":  "output/Forms_with_QR",  // QR PDF output folder
  "forms_mixed_input_folder": "input/Forms_mixed",    // Mixed-form PDF input folder
  "forms_mixed_output_folder":"output/Forms_mixed",   // Mixed-form PDF output folder
  "form_selector_file": "config/form_selector.yaml",  // Classifier prompt (default: config/form_selector.yaml)
  "forms_config_dir": "config/forms",
  "debug": false,                    // true = keep temp images and PDF in input folder
  "render_dpi": 200,                 // Default DPI for rendering image if DPI information not found in PDF
  "log_file":         "logs/pipeline.log",         // All log messages
  "log_warning_file": "logs/pipeline_warnings.log",// WARNING and ERROR messages only
  "process_log_file": "logs/process_log.jsonl",    // Machine-readable record of each processed PDF
  "llm": {
    "sys_prompt": "...",             // System prompt sent to the LLM on every call
    "default_user_prompt": "...",    // User prompt used when a page has no prompt defined
    "endpoint": "https://openrouter.ai/api/v1",
    "model": "qwen/qwen3.5-9b",     // Any OpenRouter model ID
    "api_key": "sk-or-v1-...",       // Your OpenRouter API key
    "timeout": 60,                   // LLM request timeout in seconds
    "params": {
      "temperature": 0.01
    }
  }
}
```

At least one input/output folder pair must be defined. Each input key must be paired with its output key: `forms_input_folder`/`forms_output_folder`, `forms_qr_input_folder`/`forms_qr_output_folder`, and `forms_mixed_input_folder`/`forms_mixed_output_folder`.


### Form config — `config/forms/<FORM_ID>.json`

One file per form type. Copy `config/forms/EXAMPLE_FORM.json` as a starting point.

```jsonc
{
  "form_id": "F001",
  "description": "Optional description",  // Optional
  "language": "en",                       // Optional
  "num_pages": 2,                      // Required for Single-form and mixed PDFs
  "excel_file": "output/F001.xlsx",    // Optional - path to the output Excel file
  "plugin_create_record": "plugin/create_record.py -i <json>",   // Optional - see below
  "plugin_update_record": "plugin/update_record.py -i <json>",   // Optional - see below
  "plugin_process_file":  "plugin/process_file.py -i <pdf>",      // Optional - see below
  "pages": [
    {
      "page": 1,                        // 1-indexed page number within the form
      "prompt": "...",                  // Optional — overrides default_user_prompt
      "output_format": "json",          // Optional — "json" (default) or "text". 
      "output_key": "text",             // Optional — key name for text output (default "text"); only used when output_format is "text"
      "schema": [                       // List of fields to extract (used when output_format is "json")
        {
          "key": "patient_name",
          "type": "text",               // text | boolean | select | multiple
          "remark": "Get the full name of the patient",
          "description":  "Full name"    // optional: used for display in Review GUI, not passed to LLM
        },
        {
          "key": "gender",
          "type": "select",
          "options": ["Male", "Female", "Other"],
          "remark": "Circle the patient's gender",
          "description":  "Gender"    // optional: used for display in Review GUI, not passed to LLM
        }
      ]
    }
  ]
}
```

**Field types:**

| Type | LLM output |
|---|---|
| `text` | Extracted string |
| `boolean` | `true` or `false` |
| `select` | Exactly one value from `options` |
| `multiple` | Comma-separated values from `options` |

If `prompt` is omitted from a page, `default_user_prompt` from the global config is used.

**`output_format`** controls how the LLM response is stored for a page:

| Value | Behaviour |
|---|---|
| `"json"` (default) | LLM returns structured JSON matched to the `schema` fields |
| `"text"` | LLM response is stored as a plain string under the key specified by `output_key` (default `"text"`).  |



**Plugin hooks** — three optional keys trigger an external script at lifecycle events:

| Key | Fires when | Placeholder |
|---|---|---|
| `plugin_create_record` | After a JSON/Excel record is written during batch processing | `<json>` → absolute path of the JSON output file |
| `plugin_update_record` | After a record is saved in the Review app | `<json>` → absolute path of the JSON output file |
| `plugin_process_file` | After a PDF finishes processing and is moved to `ORIGINAL_PDFs/` | `<pdf>` → absolute path of the moved PDF |

The value is a command template. `<json>` or `<pdf>` is replaced with the actual file path before the command is run. If the first token ends with `.py` it is invoked with the current Python interpreter automatically.


Plugin failures are logged as warnings and never interrupt the pipeline. `plugin_process_file` fires once per PDF even when the PDF contains multiple form instances; `plugin_create_record` fires once per form instance.


---

### Form selector config — `config/form_selector.yaml`

Required when using the mixed-form mode. Contains two prompts sent to the LLM for every page:

```yaml
sys_prompt: |
  You are a form classification assistant. Return ONLY a valid JSON object.

user_prompt: |
  Examine the form image and determine which form type it belongs to.
  This will always be the first page of a new form instance.

  Available form types:
  - form_id: "F001"
    Identify by: Title reads "Annual Report"; fields include Name, ID, Date

  - form_id: "F002"
    Identify by: Title reads "Monthly Summary"; table with columns: Month, Amount

  Return a JSON object with exactly one key:
    form_id — one of the form_id values listed above, or null if unknown

  Example output: {"form_id": "F001"}
```

- `form_id` values must match filenames in `config/forms/` (without the `.json` extension).
- A template is provided at `config/form_selector.yaml` — edit the form list to match your actual forms.


---

## Usage

```bash
# Run the pipeline (processes all PDFs in both input folders):
dhocr-batch

# Specify a different global config file:
dhocr-batch --config path/to/other_config.json
```

---

## Review App

A browser-based GUI for inspecting and correcting extracted data:

```bash
# Use the default config (config/global_config.json):
dhocr-review

# Specify a different global config file:
dhocr-review --config path/to/other_config.json
```

The app opens automatically at `http://localhost:5187`. It reads the process log to list processed PDFs. For each record you can:

- Browse the PDF pages side-by-side with the extracted fields.
- Edit any field value and save corrections back to the JSON file.
- Filter records by date range.
- Resize the sidebar and the PDF/fields panels by dragging the splitters.

---



## Output

Output mirrors the input subfolder structure:

```
output/
  Forms/
    FHS-Q1-TC-v1/
      batch1/
        scan_001_p1-2_FHS-Q1-TC-v1.json   # Extracted data per form instance
      ORIGINAL_PDFs/
        scan_001.pdf                       # Original PDF moved here after processing
  Forms_with_QR/
    folder1/
      scan_002_p1-3_F001.json
      ORIGINAL_PDFs/
        scan_002.pdf
  Forms_mixed/
    scan_003_p1-2_F001.json               # Instance of F001 found within the mixed PDF
    scan_003_p3-3_F002.json               # Instance of F002 found within the same PDF
    ORIGINAL_PDFs/
      scan_003.pdf
  <FORM_ID>.xlsx                          # One Excel file per form type (appended each run)
logs/
  pipeline.log                            # Full log from all runs
  pipeline_warnings.log                   # Warnings and errors only
  process_log.jsonl                       # One JSON record per processed PDF
```

Each JSON file contains a flat key-value dict. If the same key appears on multiple pages, the values are merged into a list.

### Naming convention of output JSON
**Naming:** `{original_stem}_p{first_page}-{last_page}_{Form ID}.json`

- `{original_stem}` — filename of the source PDF without extension
- `{first_page}` and `{last_page}` — page numbers within the **original** PDF (1-indexed) that belong to this form instance
- `{Form ID}` - form ID of the PDF pages


**Example:** A scan `scan_001.pdf` containing two back-to-back instances of a 2-page form produces:

```
output/Forms/FHS-Q1-TC-v1/
  scan_001_p1-2_FHS-Q1-TC-v1.json   ← instance 1 data
  scan_001_p3-4_FHS-Q1-TC-v1.json   ← instance 2 data
  ORIGINAL_PDFs/
    scan_001.pdf                           ← original moved here after processing
```

If the same output file already exists it is overwritten.

### Process log

The process log (`process_log.jsonl`) records one line per processed PDF in JSON Lines format:

```json
{"timestamp": "2026-05-01T10:30:00", "original_pdf": "input/Forms/...", "instances": [
  {"form_id": "FHS-Q1-TC-v1",  "json": "output/.../scan_001_p1-2_FHS-Q1-TC-v1.json"}
  {"form_id": "FHS-Q1-TC-v1",  "json": "output/.../scan_001_p3-4_FHS-Q1-TC-v1.json"}
]
}
```

When `debug` is `true`, the PDF stays in the input folder and temp images are preserved in `temp/`.
