# Manufacturer Copay / Savings Program Extractor (GoodRx + Web + PDF)

This project takes a list of **brand drugs** from an Excel workbook and attempts to extract **manufacturer copay / savings / assistance program** information. It stores normalized results in **SQLite**, and (when possible) stores a **full structured program schema** extracted from manufacturer pages and PDFs.

The pipeline is designed to be resilient against:
- “JS shell” pages that require rendering
- bot walls / soft blocks
- pages with weak navigation
- PDFs containing the real terms

---

## What it does

For each drug in the Excel input (`Database_Send (2).xlsx`):

1. **GoodRx manufacturer modal (primary path)**
   - Visits `https://www.goodrx.com/<drug-name>`
   - Clicks the **Manufacturer** section
   - Scrapes the modal fields:
     - Program name
     - Offer text (e.g., “Pay as little as …”)
     - Phone number
     - Website (CTA)

2. **Full schema extraction (2-pass)**
   - Starts from the manufacturer website URL found in GoodRx (if present)
   - Adds additional candidate URLs via **DuckDuckGo**
   - Ranks URLs using heuristics (drug-token matching, program keywords, domain scoring)
   - Tries to extract a **full JSON schema**:
     - Uses `crawl4ai_fetch` first (fast)
     - Falls back to Selenium rendering if blocked/blank/shell-like
     - If a PDF is detected, uses PyMuPDF/pdfplumber + AI extraction

3. **Fallback path if GoodRx modal fails**
   - Uses `co-pay.com` search + activation link extraction (Selenium)
   - If that fails, uses DuckDuckGo candidate selection + schema extraction

4. **Post-processing enforcement**
   - Reduces results to **exactly one program per drug** (deterministic)
   - Drops empty extractions
   - Optionally can drop “discount_card-only” outputs (configurable)

5. Saves results to SQLite:
   - `manufacturer_coupons` table for human-facing fields
   - `ai_page_extractions` table for full extracted JSON schema

---

## Outputs

### SQLite DB: `goodrx_coupons.db`

#### Table: `manufacturer_coupons`
Stores the “business fields” for each drug:
- `drug_name`
- `program_name`
- `manufacturer_url`
- `offer_text`
- `phone_number`
- `confidence` (e.g. `GoodRx`, `fallback-copay`, `SE - ai-extracted`)
- `has_copay_program` (0/1)
- `last_extracted_at` (UTC ISO8601)
- `extraction_log` (debug breadcrumb trail)

#### Table: `ai_page_extractions`
Stores **the final normalized full schema JSON** per drug:
- `drug_name` (PK)
- `ai_extraction` (JSON string)

---

## Full Schema (high-level)

The schema is a JSON array with one object, containing:
- `drug` (name, manufacturer, etc.)
- `programs[]` (copay / pap / foundation / rebate / bridge_fill / etc.)
- `sources[]` (URLs used, content types, fields supported)
- `summary` fields

**Important enforcement:** the system reduces to **one best program** via:
- type priority (copay > pap > others)
- confidence tier priority (A > B > C > …)
- actionability (direct enrollment/download links, PDFs)
- completeness (presence of TLDRs, eligibility, contact, CTA)

---

## Requirements

### Python
- Python 3.9+ recommended

### Dependencies
Core:
- `openpyxl`
- `selenium`
- `python-dotenv`
- `requests`
- `openai` (or compatible OpenAI python SDK)

PDF (at least one):
- `PyMuPDF` (`fitz`) **or**
- `pdfplumber`

Custom module:
- `crawl4ai_fetch.py` must exist and export `crawl4ai_fetch(url, timeout_s=...)`

Chrome:
- Google Chrome installed
- Matching ChromeDriver available on PATH (or Selenium Manager working)

---

## Installation

```bash
python -m venv .venv
source .venv/bin/activate  # (Windows: .venv\Scripts\activate)

pip install -U pip
pip install openpyxl selenium python-dotenv requests openai PyMuPDF pdfplumber
