# Academic ETL Pipeline

A full end-to-end pipeline for collecting, processing, and storing structured academic program data from university websites. The system is composed of three stages: **Web Crawling**, **Content Extraction**, and **ETL (Extract-Transform-Load)** into MongoDB using a local LLM.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Components](#components)
  - [Stage 1 – Web Crawler (Web_Crawler.ipynb)](#stage-1--web-crawler-web_crawleripynb)
  - [Stage 2 – Enhanced Web Crawler (Web_Crawlerv2.ipynb)](#stage-2--enhanced-web-crawler-web_crawlerv2ipynb)
  - [Stage 3 – ETL Pipeline (ETL_Pipeline_v2.ipynb)](#stage-3--etl-pipeline-etl_pipeline_v2ipynb)
- [Data Models](#data-models)
- [Configuration Reference](#configuration-reference)
- [Dependencies](#dependencies)
- [Setup & Usage](#setup--usage)
- [Data Flow Diagram](#data-flow-diagram)
- [Output](#output)
- [Error Handling & Retry Logic](#error-handling--retry-logic)
- [Limitations & Notes](#limitations--notes)

---

## Overview

This project automates the collection of academic program information from university websites and stores it in a structured MongoDB database. It is designed to run in **Google Colab** and relies on a locally served **Ollama LLM (LLaMA 3)** for both intelligent URL filtering and structured data extraction.

The full pipeline runs as three sequential stages:

```
University Website
       │
       ▼
[Web Crawler] ──► Raw HTML pages
       │
       ▼
[Text Extractor] ──► Clean .txt files (per page)
       │
       ▼
[ETL Pipeline (LLM + Pydantic)] ──► Structured JSON
       │
       ▼
[MongoDB] ──► institutions + programs collections
```

---

## Architecture

| Stage | Notebook | Input | Output |
|-------|----------|-------|--------|
| Web Crawling | `Web_Crawler.ipynb` | Website URL | Cleaned `.txt` files |
| Enhanced Crawling | `Web_Crawlerv2.ipynb` | Website URL | Cleaned `.txt` files (threaded) |
| ETL & Load | `ETL_Pipeline_v2.ipynb` | `.txt` files | MongoDB documents |

---

## Components

### Stage 1 – Web Crawler (`Web_Crawler.ipynb`)

A modular academic web crawler that discovers, filters, downloads, and extracts readable content from university websites.

#### Pipeline Steps

1. **Setup** – Install required packages and start the Ollama server.
2. **Configure** – Set the target URL and crawl parameters.
3. **Discover URLs** – Recursively crawl the website to find all internal links.
4. **Filter URLs (AI)** – Use Ollama LLM to select only academically relevant pages.
5. **Download Content** – Fetch the full HTML of each selected URL in parallel.
6. **Extract Text** – Strip HTML tags, clean whitespace, and save to `.txt` files.

#### URL Discovery

The crawler starts from a `START_URL` and performs a recursive depth-first crawl up to `MAX_DEPTH` levels, collecting all internal links. Each discovered URL is normalized (fragments stripped, query parameters removed, trailing slashes removed) and validated.

**Rejected URL patterns** include:
- Static assets: `css`, `js`, `images`, `fonts`, `pdf`, `media`
- Auth pages: `login`, `logout`, `register`, `portal`, `sso`
- News/Events: `blog`, `news`, `event`, `gallery`, `announcement`
- Careers: `jobs`, `career`, `recruitment`
- Social/Legal: `privacy`, `terms`, `facebook`, `twitter`
- Admin/System: `admin`, `api`, `wp-admin`, `config`

**Rejected file extensions**: `.jpg`, `.png`, `.gif`, `.svg`, `.css`, `.js`, `.pdf`, `.zip`, `.mp4`, `.docx`, `.xlsx`, and more.

#### AI URL Filtering

After discovery, URLs are passed in batches to the Ollama LLM (batch size: 20). The LLM selects URLs that are likely to contain academic content, guided by:

**Include keywords**: `program`, `course`, `degree`, `undergraduate`, `postgraduate`, `faculty`, `department`, `admission`, `curriculum`, `syllabus`, `diploma`, `bachelor`, `master`, `phd`

**Exclude keywords**: `news`, `event`, `staff`, `library`, `login`, `gallery`, `download`, `contact`

A heuristic pre-filter is also applied before the AI call to reduce unnecessary LLM usage.

#### Text Extraction

For each downloaded page:
- Tags removed: `<script>`, `<style>`, `<nav>`, `<footer>`, `<aside>`, `<header>`, `<iframe>`, `<form>`
- HTML comments stripped
- Text extracted with newline separators
- Multiple whitespace collapsed
- Pages with fewer than 500 characters are skipped

#### Configuration

```python
START_URL            = "https://www.bms.ac.lk/"
MAX_DEPTH            = 100
MAX_DOWNLOAD_WORKERS = 32
AI_BATCH_SIZE        = 20
AI_MAX_WORKERS       = 10
MIN_TEXT_LENGTH      = 500
OUTPUT_DIR           = "/content/drive/MyDrive/academic_content_output/{domain}"
```

---

### Stage 2 – Enhanced Web Crawler (`Web_Crawlerv2.ipynb`)

An improved version of Stage 1 with multi-threaded crawling, a worker queue architecture, and more robust AI filtering with automatic retry and server recovery.

#### Key Improvements Over Stage 1

| Feature | Stage 1 | Stage 2 |
|---------|---------|---------|
| Crawl threading | Single-thread recursive | Worker pool (up to 100 threads) |
| Task distribution | Recursive call stack | Thread-safe `Queue` |
| Program tree | Not tracked | Parent-child link tree |
| AI filter retry | None | Up to 3 retries with server restart |
| Backoff strategy | None | Exponential: `2 × (attempt + 1)` seconds |
| LLM failure recovery | None | Auto-restarts Ollama + re-pulls model |

#### Threaded Crawl Architecture

```
task_queue (Queue)
    │
    ├── worker thread 1 ──► extract_program_links() ──► add new URLs to queue
    ├── worker thread 2 ──►      ...
    └── worker thread N ──►      ...
```

Global thread-safe state is managed via `threading.Lock()` for both `visited` URLs and the `program_tree` structure. `task_queue.join()` blocks until all URLs have been processed.

#### Program Tree

The crawler maintains a hierarchical map of page relationships:

```python
program_tree = {
    "https://mgmt.cmb.ac.lk/programs": [
        "https://mgmt.cmb.ac.lk/programs/bsc-accounting",
        "https://mgmt.cmb.ac.lk/programs/msc-finance"
    ],
    ...
}
```

#### Enhanced AI Filtering Prompt

The LLM prompt explicitly lists URLs to **include**:
- Program listings (undergraduate/postgraduate)
- Degree information pages (BSc, MSc, PhD)
- Course catalogs and unit descriptions
- Admission requirements and eligibility
- Curriculum/syllabus pages
- Academic prospectus

And URLs to **exclude**:
- Individual staff profiles
- Research projects/publications
- News articles and events
- Student clubs/societies/field trips
- Facilities/museums/libraries
- Alumni/awards/scholarships/contact pages
- Date-patterned URLs (e.g. `/2018/05/23/`)
- Image galleries

#### Configuration

```python
START_URL            = "https://mgmt.cmb.ac.lk/mgmt_department-of-accounting"
MAX_DEPTH            = 50
MAX_WORKERS          = 100
MAX_DOWNLOAD_WORKERS = 10
AI_BATCH_SIZE        = 10
AI_MAX_WORKERS       = 5
MIN_TEXT_LENGTH      = 500
OUTPUT_DIR           = "/content/drive/MyDrive/academic_content_output/{domain}"
```

---

### Stage 3 – ETL Pipeline (`ETL_Pipeline_v2.ipynb`)

Reads the `.txt` files produced by the crawlers, uses Ollama LLM to extract structured academic program data from each file, validates it with Pydantic, and loads it into MongoDB.

#### Pipeline Steps

1. **Setup** – Install Ollama (if running in Colab), configure dependencies.
2. **Define Models** – Pydantic schemas for `Institution` and `Program`.
3. **Configure** – Set Ollama and MongoDB connection details.
4. **Run ETL** – For each `.txt` file in `./data`:
   - Send text to Ollama with a structured extraction prompt.
   - Parse and validate the LLM response as JSON.
   - Retry up to 3 times on parse or validation failures.
   - Insert valid records into MongoDB.
5. **Summary** – Print statistics on processed, skipped, and failed files.

#### LLM Extraction

The LLM is instructed to act as an expert data extraction AI:
- Returns a strict JSON object conforming to the `Program` schema
- Rejects non-program content with `{"not_relevant": true}`
- Never uses ellipsis or placeholder values
- Puts unrecognized fields in the `extensions` object
- Uses only field names from the schema (e.g. never `"program_name"` or `"programmes"`)

A low temperature (`0.1`) is used to maximize determinism in structured extraction.

#### Retry & Repair Logic

| Failure Type | Recovery Action |
|---|---|
| JSON parse error | Attempt to repair by balancing braces; retry with error context |
| Pydantic validation error | Feed the exact validation error back to the LLM on next attempt |
| `confidence_score < 0.5` | Skip the record (mark as low confidence) |
| `not_relevant: true` | Skip the file entirely |

Maximum attempts per file: **3** (initial + 2 retries).

#### MongoDB Storage

**Institution document** (created once per pipeline run):

```json
{
  "name": "BMS Campus",
  "institution_code": "INST-20240101",
  "type": ["University"],
  "country": "Sri Lanka",
  "programs": [],
  "created_at": "2024-01-01T00:00:00"
}
```

**Program document** (one per extracted program):

```json
{
  "institution_id": "<ObjectId>",
  "name": "BSc in Business Management",
  "program_code": "BSC-BM-001",
  "level": "Undergraduate",
  "duration": { "years": 3 },
  "delivery_mode": ["Full-time"],
  "fees": { "amount": 250000, "currency": "LKR", "period": "per year" },
  "eligibility": { "minimum_qualifications": "3 A/L passes" },
  "confidence_score": 0.92,
  "source_file": "programs_page.txt",
  "extracted_at": "2024-01-01T10:00:00"
}
```

#### Configuration

```python
CONFIG = {
    "OLLAMA_BASE_URL":          "http://localhost:11434",
    "OLLAMA_MODEL":             "llama3",
    "MONGODB_URI":              "",          # Set your MongoDB connection string
    "DATABASE_NAME":            "education_db",
    "COLLECTION_INSTITUTIONS":  "institutions",
    "COLLECTION_PROGRAMS":      "programs",
    "DATA_FOLDER":              "./data",
    "MAX_RETRIES":              2,
    "TIMEOUT":                  60
}
```

---

## Data Models

### Institution

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | `str` | ✅ | Official institution name |
| `institution_code` | `str` | ❌ | Unique identifier (auto-generated: `INST-YYYYMMDD`) |
| `description` | `str` | ❌ | Brief overview of the institution |
| `type` | `List[str]` | ❌ | e.g. `["University", "Private"]` |
| `country` | `str` | ❌ | Defaults to `"Sri Lanka"` |
| `website` | `str` | ❌ | Official website URL |
| `recognition` | `Dict` | ❌ | Accreditation and recognition details |
| `contact_info` | `Dict` | ❌ | Address, phone, email |
| `confidence_score` | `float` | ✅ | LLM confidence (0.0 – 1.0) |

### Program

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | `str` | ✅ | Full program name |
| `program_code` | `str` | ❌ | Unique program identifier |
| `description` | `str` | ❌ | Program overview |
| `level` | `str` | ❌ | `Undergraduate`, `Postgraduate`, `Diploma`, `Certificate`, `Foundation` |
| `duration` | `Dict` | ❌ | `{"years": 3, "months": 0, "weeks": 0}` |
| `delivery_mode` | `List[str]` | ❌ | `Full-time`, `Part-time`, `Online`, `Hybrid`, `Weekend` |
| `fees` | `Dict` | ❌ | `{"amount": 0, "currency": "LKR", "period": "per year", "breakdown": "..."}` |
| `eligibility` | `Dict` | ❌ | `{"minimum_qualifications": "...", "gpa": 0.0, "age_limit": 0, "work_experience": "..."}` |
| `curriculum_summary` | `str` | ❌ | Summary of subjects/modules |
| `specializations` | `List[str]` | ❌ | Available tracks or majors |
| `url` | `str` | ❌ | Source URL for the program |
| `extensions` | `Dict` | ❌ | Any extra data not captured by other fields |
| `confidence_score` | `float` | ✅ | LLM confidence (0.0 – 1.0) |

---

## Dependencies

| Package | Used In | Purpose |
|---------|---------|---------|
| `ollama` / `langchain-ollama` | All | Local LLM inference (LLaMA 3) |
| `pydantic` | ETL Pipeline | Data validation and schema enforcement |
| `pymongo` | ETL Pipeline | MongoDB connection and CRUD |
| `requests` | All | HTTP requests |
| `beautifulsoup4` | Web Crawlers | HTML parsing |
| `lxml` | Web Crawlers | Fast HTML/XML parsing backend |
| `tqdm` | Web Crawlers | Progress bars |
| `pathlib` | ETL Pipeline | File path operations |
| `concurrent.futures` | All | Thread pool execution |
| `queue` | Web Crawlerv2 | Thread-safe task queue |
| `threading` | Web Crawlerv2 | Lock management |

All notebooks install their dependencies at runtime using `pip install`.

---

## Setup & Usage

### Prerequisites

- Python 3.10+
- [Ollama](https://ollama.ai/) installed and accessible at `http://localhost:11434`
- LLaMA 3 model pulled: `ollama pull llama3`
- A running MongoDB instance (local or Atlas)
- Google Drive mounted (if saving crawler output to Drive)

### Running the Full Pipeline

#### Step 1: Crawl a University Website

Open `Web_Crawlerv2.ipynb` (recommended) or `Web_Crawler.ipynb`.

1. Set the `START_URL` to the target university's program page.
2. Adjust `MAX_DEPTH`, `MAX_WORKERS`, and `OUTPUT_DIR` as needed.
3. Run all cells. Output `.txt` files will be saved to `OUTPUT_DIR`.

#### Step 2: Run the ETL Pipeline

Open `ETL_Pipeline_v2.ipynb`.

1. Set `CONFIG["MONGODB_URI"]` to your MongoDB connection string.
2. Place the `.txt` files from Step 1 into the `./data` folder.
3. Run all cells. When prompted, enter the institution name.
4. Structured program records will be inserted into MongoDB.

---

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        University Website                        │
│                  (e.g., https://www.bms.ac.lk/)                 │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                        HTTP GET requests
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                     URL Discovery (Crawler)                      │
│  • Recursive / multi-threaded crawl                             │
│  • Normalize, deduplicate, reject non-content URLs              │
│  Output: ~30-100 raw URLs                                       │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                     Heuristic + LLM filter
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AI URL Filtering (Ollama)                     │
│  • Batch URLs → LLM → keep only academic program pages          │
│  Output: ~10-30 relevant URLs                                   │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                    Parallel HTTP downloads
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    HTML → Text Extraction                        │
│  • Strip nav/footer/scripts/ads                                 │
│  • Clean whitespace, enforce min length                         │
│  Output: One .txt file per page                                 │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                  Place .txt files in ./data folder
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    LLM Extraction (Ollama)                       │
│  • Prompt: "Extract academic program data as JSON"              │
│  • Temperature: 0.1 (deterministic)                             │
│  • Retry with error feedback on failure                         │
│  Output: Validated Pydantic Program objects                     │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                    PyMongo insert operations
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                         MongoDB Atlas                            │
│  • education_db.institutions  (one per run)                     │
│  • education_db.programs      (one per extracted program)       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Output

### MongoDB Collections

**`education_db.institutions`**
```json
{
  "_id": "ObjectId(...)",
  "name": "BMS Campus",
  "institution_code": "INST-20240101",
  "type": ["University", "Private"],
  "country": "Sri Lanka",
  "website": "https://www.bms.ac.lk",
  "programs": [],
  "created_at": "2024-01-01T09:00:00"
}
```

**`education_db.programs`**
```json
{
  "_id": "ObjectId(...)",
  "institution_id": "ObjectId(...)",
  "name": "BSc (Hons) in Business Management",
  "level": "Undergraduate",
  "duration": { "years": 3, "months": 0 },
  "delivery_mode": ["Full-time"],
  "fees": {
    "amount": 250000,
    "currency": "LKR",
    "period": "per year"
  },
  "eligibility": {
    "minimum_qualifications": "3 passes at G.C.E. A/L"
  },
  "curriculum_summary": "Covers core business subjects including marketing, accounting, HRM, and strategy.",
  "confidence_score": 0.91,
  "source_file": "bms_ac_lk_programs.txt",
  "extracted_at": "2024-01-01T09:15:00"
}
```

### Text Files (Crawler Output)

Each `.txt` file contains the source URL header followed by the cleaned page text:

```
URL: https://www.bms.ac.lk/programmes/bsc-business-management
================================================================================
BSc (Hons) in Business Management

Duration: 3 Years | Mode: Full-time | Fees: LKR 250,000/year

Entry Requirements
...
```

---

## Error Handling & Retry Logic

### ETL Pipeline

| Scenario | Handling |
|----------|----------|
| Ollama not running | Raises connection error with clear message |
| MongoDB unreachable | Raises `ConnectionFailure` at startup |
| LLM returns malformed JSON | Attempts brace-balancing repair; retries |
| LLM returns invalid schema | Passes validation errors back to LLM; retries |
| `confidence_score < 0.5` | Record skipped, logged as low confidence |
| `{"not_relevant": true}` | File skipped, counted in statistics |
| Max retries exceeded | File logged as failed; pipeline continues |

### Web Crawlers

| Scenario | Handling |
|----------|----------|
| HTTP request timeout | Caught per-URL; URL marked as failed |
| HTML parse error | BeautifulSoup fallback; empty result returned |
| Ollama connection refused (v2) | Auto-restart Ollama server; exponential backoff |
| LLM JSON parse failure (v2) | Retry up to 3 times; fall back to empty list |
| Duplicate URLs | `visited` set prevents re-processing |

---

## Limitations & Notes

- **LLM accuracy**: Extraction quality depends on the source text quality and LLM capability. `confidence_score` provides a self-reported quality indicator but may not always be reliable.
- **Ollama must be running**: The ETL pipeline and crawlers require a local Ollama server. In Colab, the notebooks install and start Ollama automatically.
- **Rate limiting**: Web crawlers include a 0.5-second delay between AI filter batches to avoid overloading the local LLM server.
- **Data folder**: The ETL pipeline expects `.txt` files to already be present in `./data`. Run one of the crawlers first to populate this folder.
- **Sri Lanka focus**: The default `country` field is set to `"Sri Lanka"` in the Institution model, reflecting the intended use case for local universities.
- **Google Colab**: All notebooks are designed to be run in Google Colab with Google Drive mounted for persistent storage of crawled content.
