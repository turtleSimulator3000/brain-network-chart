---
name: dicom2bids
description: A skill to convert DICOM files to BIDS format with AI-powered series classification.
---

# DICOM2BIDS Conversion Skill

## Purpose

Automate conversion of DICOM MRI files to BIDS (Brain Imaging Data Structure) format using intelligent AI-powered series classification. This skill analyzes neuroimaging datasets, discovers layout structure, classifies MRI sequences, and generates properly organized BIDS output with minimal manual intervention.

## What This Skill Does

1. **Dataset Discovery**: Analyzes directory structure to detect subject/session organization (modality-grouped vs subject-toplevel)
2. **DICOM Metadata Reading**: Extracts series descriptions and acquisition parameters
3. **AI Series Classification**: Uses Ollama LLM (qwen3) to intelligently classify MRI sequences into BIDS datatypes/suffixes
4. **Configuration Generation**: Creates dcm2bids config.json automatically
5. **DICOM Conversion**: Runs dcm2bids for each subject/session with proper handling
6. **Post-Processing**: Fixes BIDS compliance issues (fmap echoes, ASL metadata, sbref bval/bvec)
7. **Report Generation**: Creates HTML report summarizing conversion results

## Execution

Run the script with optional input and output directories:

```bash
python3 dicom2bids_agent.py [data_dir] [output_dir]
```

### Parameters

- **data_dir** (optional): Path to directory containing DICOM files or archives (.zip/.tgz)
  - Default: `/ram/USERS/tao/code/new/data`
  - Supports nested subject/session structures
  
- **output_dir** (optional): Path where BIDS-formatted data will be saved
  - Default: `/ram/USERS/tao/code/new/bids_output`
  - Creates subdirectories: `sub-*/`, `tmp_dcm2bids/`, config files, reports

### Examples

Convert with defaults:
```bash
python3 dicom2bids_agent.py
```

Convert from custom directory:
```bash
python3 dicom2bids_agent.py /data/my_study /output/bids_dataset
```

## Required Dependencies

- **Python 3.8+** with packages:
  - `pydicom` - DICOM file reading
  - `httpx` - Async HTTP client for Ollama API
  - `asyncio` - Concurrent processing
  
- **External tools**:
  - `dcm2bids` - DICOM to BIDS converter
  - `dcm2niix` - DICOM to NIfTI converter (required by dcm2bids)
  - `Ollama` server running with `qwen3:latest` model

### Environment Variables

- `DICOM2BIDS_MODEL` - Ollama model name (default: `qwen3:latest`)

## Ollama Configuration

The script connects to Ollama servers at:
- `localhost:11434` (default local)
- `yukon.acm.unc.edu:11434` (fallback)

Ensure Ollama is running with the qwen3 model available. The script uses:
- Zero temperature (deterministic classification)
- ~250 token max output per classification
- 30-second timeout per Ollama call
- Concurrent classification (8 requests at a time)

## Input Dataset Structure

The script supports multiple dataset layouts:

### Modality-Grouped Layout
```
data_dir/
├── T1W/
│   ├── sub-001/
│   │   └── [DICOM files]
│   └── sub-002/
├── DTI/
│   ├── sub-001/
│   └── sub-002/
```

### Subject-TopLevel Layout
```
data_dir/
├── sub-001/
│   ├── T1W/
│   ├── DTI/
│   └── fMRI/
├── sub-002/
```

### Session Support
Automatically detects date-based or named sessions:
```
sub-001/
├── 2024-01-15/
│   └── [DICOM files]
└── 2024-03-20/
```

## Output Structure

BIDS-organized directory:
```
output_dir/
├── sub-001/
│   ├── anat/
│   │   ├── sub-001_T1w.nii.gz
│   │   └── sub-001_T1w.json
│   ├── func/
│   │   ├── sub-001_task-rest_bold.nii.gz
│   │   └── sub-001_task-rest_bold.json
│   └── dwi/
│       └── [DWI files]
├── sub-002/
├── dataset_description.json
├── dcm2bids_config.json
├── conversion_report.html
└── tmp_dcm2bids/
```

## Classification Logic

### Pre-Checks (Before LLM)
High-confidence patterns that bypass LLM:
- **Skip**: Localizers, scouts, derived maps (ADC, FA, TRACE), auxiliary sequences
- **dwi/sbref**: b=0-only acquisitions
- **perf/asl**: Arterial spin labeling sequences
- **anat/FLAIR**: FLAIR sequences
- **func/bold rest**: Resting-state fMRI

### LLM Classification
For ambiguous sequences, the LLM receives:
- SeriesDescription
- MRAcquisitionType
- EchoTime (ms)
- RepetitionTime (ms)
- ParentDir hint (modality-named directory)

Output: `{skip: bool, datatype: str, suffix: str, task: str, reason: str}`

### Fallback (If LLM Fails)
Regex-based classification rules (hardcoded fallback ensures robustness)

## Post-Processing Operations

1. **Field Map Echo Fixing** (`fix_fmap_echoes`):
   - Renames dcm2bids multi-run phasediff to proper BIDS fmap names
   - Classifies using EchoNumber and ImageType
   - Injects EchoTime1/EchoTime2

2. **SBREF Cleanup** (`fix_sbref_bval`):
   - Removes invalid `*_sbref.bval` and `*_sbref.bvec` files

3. **ASL Metadata** (`fix_asl_metadata`):
   - Adds required BIDS ASL fields (PostLabelingDelay, etc.)

## Output Report

Generates `conversion_report.html` with:
- Conversion summary per subject/session
- Series classification results
- DICOM→BIDS pairings
- Errors and warnings
- Unmatched series

## Error Handling

- Gracefully handles missing DICOM files
- Retries failed Ollama calls with fallback regex
- Skips subjects with conversion errors
- Cleans up temporary files
- Preserves BIDS output on partial failures

## UTF-8 Support

Automatically configures UTF-8 output on Windows to display special characters (→, ✓, etc.).
