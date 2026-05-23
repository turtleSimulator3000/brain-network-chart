---
name: dicom2bids
description: 'Convert DICOM neuroimaging data to BIDS format using dicom2bids_agent.py. Use when user wants to run DICOM to BIDS conversion, convert MRI scans, run dicom2bids, process neuroimaging data, or convert brain scan files. Checks and installs all required tools, sets up Ollama, prompts for directories, runs the conversion, and summarizes results.'
argument-hint: 'Optional: input DICOM directory and output BIDS directory'
---

# DICOM → BIDS Conversion (dicom2bids_agent.py)

This skill guides a non-developer user through the full process of running `dicom2bids_agent.py`
— from checking prerequisites to viewing the final report. No coding knowledge is required.

---

## When to Use
- User wants to convert DICOM files to BIDS format
- User mentions "run dicom2bids", "convert MRI scans", "BIDS conversion", or similar
- User wants to process neuroimaging / brain scan data

---

## Procedure

Work through each phase in order. Explain what you are doing in plain language.
Never modify any source code files.

---

### Phase 1 — Check and install prerequisites

Run each check in the terminal. If a tool is missing, install it automatically.

#### 1.1 — Python 3.11+

```bash
python3 --version
```

- If the output is `Python 3.11.x` or higher → OK.
- If missing or older, check for conda:

```bash
conda --version
```

If conda is available, create the environment:

```bash
conda create -y -n brainchart python=3.11
conda activate brainchart
```

If conda is not available, tell the user:
> "Python 3.11 or newer is required. Please install Miniconda from https://docs.conda.io/en/latest/miniconda.html and re-run this skill."

#### 1.2 — Python packages (httpx, pydicom, dcm2bids, dcm2niix)

The fastest way is to use the provided install script, which handles everything:

```bash
bash install.sh
```

If `install.sh` is not available or fails, install manually inside the conda environment:

```bash
conda activate brainchart
pip install httpx pydicom dcm2bids dcm2niix
```

Verify:

```bash
python3 -c "import httpx, pydicom; print('httpx and pydicom OK')"
dcm2bids --version
dcm2niix --version
```

If any verification fails, try:

```bash
pip install --upgrade httpx pydicom dcm2bids dcm2niix
```

#### 1.3 — Activate the environment before proceeding

```bash
conda activate brainchart
```

---

### Phase 2 — Set up Ollama

Ollama provides the local AI model that classifies MRI sequences. If it is not available,
the script falls back to built-in rules automatically (no conversion data is lost).

#### 2.1 — Check if Ollama is installed

```bash
ollama --version
```

If not found, install it (Linux):

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

For other operating systems, direct the user to: https://ollama.com/download

#### 2.2 — Check if the required model is already available

```bash
ollama list
```

If `qwen3:latest` is not listed, pull it. This downloads ~5 GB and may take a few minutes:

```bash
ollama pull qwen3:latest
```

Tell the user: "This downloads the AI model (~5 GB). Please wait — it only needs to be done once."

#### 2.3 — Start the Ollama server

First check whether it is already running:

```bash
curl -s http://localhost:11434/api/tags
```

If the command returns JSON (a list of models) → the server is already running, skip to Phase 3.

If it returns an error or hangs, start the server in the background:

```bash
ollama serve &
```

Then wait a few seconds and verify again:

```bash
sleep 3 && curl -s http://localhost:11434/api/tags
```

If Ollama still cannot start, the conversion will still run using the built-in regex fallback.
Inform the user: "Ollama could not be started. The conversion will proceed using the built-in
classification rules instead of AI. Results should still be correct for most standard sequences."

---

### Phase 3 — Confirm directories

#### 3.1 — Ask the user for paths

If the user has not already provided input and output directories, ask them:

> "Please provide two directory paths:
> 1. **Input directory** — the folder containing your DICOM files
> 2. **Output directory** — where the BIDS files should be saved
>
> Or press Enter to use the built-in defaults:
> - Input: `/ram/USERS/tao/code/new/data`
> - Output: `/ram/USERS/tao/code/new/bids_output`"

Set the variables for use in subsequent steps:

- `DATA_DIR` — user-supplied input path, or empty to use the built-in default
- `OUTPUT_DIR` — user-supplied output path, or empty to use the built-in default

#### 3.2 — Verify the input directory exists and contains DICOM files

If the user supplied a custom `DATA_DIR`:

```bash
ls "$DATA_DIR" | head -20
```

Check that the directory exists and is not empty. If it does not exist:
> "The directory `<path>` was not found. Please double-check the path and try again."

---

### Phase 4 — Run the conversion

Activate the environment, then run the script with or without custom paths.

**If the user provided both directories:**

```bash
conda activate brainchart
python3 dicom2bids_agent.py "$DATA_DIR" "$OUTPUT_DIR"
```

**If using default directories (no arguments given):**

```bash
conda activate brainchart
python3 dicom2bids_agent.py
```

**Optional overrides the user can choose:**

| Scenario | Command |
|---|---|
| Use a different AI model | `DICOM2BIDS_MODEL=llama3:latest python3 dicom2bids_agent.py ...` |
| Skip AI, use rules only | `DICOM2BIDS_MODEL=none python3 dicom2bids_agent.py ...` |

Watch the terminal output and relay progress to the user in plain language
(e.g. "Scanning DICOM files...", "Classifying sequences...", "Converting to NIfTI...").

---

### Phase 5 — Handle errors

If the script exits with an error or produces unexpected output, work through the
table below to identify the most likely cause and suggest a fix.

| Symptom | Likely cause | Suggested fix |
|---------|-------------|---------------|
| `dcm2niix: command not found` | dcm2niix not on PATH in the active environment | Run `pip install dcm2niix` in the brainchart environment |
| `ModuleNotFoundError: httpx` | httpx not installed, or wrong Python environment active | Run `conda activate brainchart && pip install httpx` |
| `PermissionError` on output dir | Output directory is read-only or does not exist | Create the directory (`mkdir -p "$OUTPUT_DIR"`) or choose a different path |
| `No DICOM files found` | Wrong input directory, or files are in a compressed format not yet extracted | Verify the input path with `ls "$DATA_DIR"` and check for `.zip` or `.tar.gz` files that need extracting |
| Ollama error / timeout | Ollama server is not running or the model is not loaded | Run `ollama serve &` and `ollama pull qwen3:latest`, then retry; or set `DICOM2BIDS_MODEL=none` to skip AI |
| `Connection refused` on Ollama | Server started but not yet ready | Wait 5–10 seconds and retry; the fallback rules will be used automatically if Ollama is unreachable |
| JSON parse error from LLM | AI model returned malformed output | The script automatically falls back to regex rules; if it keeps failing, run with `DICOM2BIDS_MODEL=none` |
| Empty output directory | dcm2bids config was generated but no sequences matched | Inspect `dcm2bids_config.json` in the output dir; the series descriptions in the data may need manual review |
| Conda environment not found | `brainchart` env was never created | Re-run `bash install.sh` or create it manually (Phase 1) |
| Out-of-memory error | Very large dataset or too many concurrent LLM calls | Close other applications to free RAM; the script limits concurrent calls to 2 by default |

If the error does not match anything above:
1. Copy the full error message from the terminal.
2. Check the log files in `<output_dir>/tmp_dcm2bids/log/` for more detail.
3. Report the error message together with the log contents for further diagnosis.

---

### Phase 6 — Review the report

After a successful run, the output directory contains:

| Path | Description |
|------|-------------|
| `<output_dir>/sub-*/` | BIDS-organised NIfTI files, one folder per subject |
| `<output_dir>/dcm2bids_config.json` | Auto-generated configuration showing how each sequence was classified |
| `<output_dir>/conversion_report.html` | Full HTML summary report |

Open the HTML report for the user:

```bash
xdg-open "$OUTPUT_DIR/conversion_report.html" 2>/dev/null \
  || echo "Report saved at: $OUTPUT_DIR/conversion_report.html"
```

If `xdg-open` prints "Report saved at..." instead of opening a browser, the environment
has no graphical display (e.g. a remote SSH session). In that case, tell the user:
> "Your system is headless — a browser cannot be launched from the terminal.
> Open the report by clicking the file in the VS Code Explorer panel:
> `<output_dir>/conversion_report.html` → right-click → **Open With…** → **Default Browser**,
> or use the **Live Server** extension to preview it."

Summarise the results in plain language, e.g.:
> "Conversion complete. Found 2 subjects, 8 functional runs, and 2 anatomical scans.
> The full report has been saved to `<output_dir>/conversion_report.html`."

---

### Phase 7 — Stop Ollama (optional)

If the user is finished and wants to free up memory:

```bash
ollama stop qwen3:latest   # unload the model from memory
sudo pkill ollama           # stop the server process
```

---

## Quick-reference: environment variable overrides

| Variable | Effect |
|----------|--------|
| `DICOM2BIDS_MODEL=qwen3:latest` | Default AI model (Ollama) |
| `DICOM2BIDS_MODEL=llama3:latest` | Use a different Ollama model |
| `DICOM2BIDS_MODEL=none` | Disable AI; use built-in regex rules only |
