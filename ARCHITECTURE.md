# ServiceNow Ticket Segregation Workflow - Architecture & Data Flow

## System Architecture Diagram

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                    SERVICENOW TICKET SEGREGATION WORKFLOW                     │
└────────────────────────────────────────────────────────────────────────────────┘

                              INPUT LAYER
                              ───────────
                    ┌────────────────────────────┐
                    │  servicenow_tickets.xlsx   │
                    │  (120 tickets)             │
                    └────────┬───────────────────┘
                             │
                    ┌────────────────────────────┐
                    │ ticket_categories.xlsx     │
                    │ (100 categories)           │
                    └────────┬───────────────────┘
                             │
                             ▼
                    ┌────────────────────────────┐
                    │   DATA PROCESSING LAYER    │
                    │   (Pandas DataFrame)       │
                    │                            │
                    │ • Load Excel files         │
                    │ • Parse data               │
                    │ • Data validation          │
                    └────────┬───────────────────┘
                             │
                             ▼
                    ┌────────────────────────────┐
                    │   TEXT PREPROCESSING       │
                    │   (Clean & Tokenize)       │
                    │                            │
                    │ • Remove special chars     │
                    │ • Convert to lowercase     │
                    │ • Split into tokens        │
                    └────────┬───────────────────┘
                             │
                             ▼
                    ┌────────────────────────────┐
                    │   ML VECTORIZATION LAYER   │
                    │   (TF-IDF + Scikit-learn)  │
                    │                            │
                    │ • TF-IDF Vectorizer        │
                    │ • Convert text → vectors   │
                    │ • N-gram support (1-2)     │
                    └────────┬───────────────────┘
                             │
                             ▼
                    ┌────────────────────────────┐
                    │   SIMILARITY MATCHING      │
                    │   (Cosine Similarity)      │
                    │                            │
                    │ For each ticket:           │
                    │ • Compare vs 100 cats      │
                    │ • Calculate similarity     │
                    │ • Select best match        │
                    └────────┬───────────────────┘
                             │
                             ▼
                    ┌────────────────────────────┐
                    │   CONFIDENCE SCORING       │
                    │                            │
                    │ Score = Similarity (0-1)   │
                    │                            │
                    │ Threshold = 0.3            │
                    │ ≥ 0.3 → Categorized        │
                    │ < 0.3 → Unknown (101st)    │
                    └────────┬───────────────────┘
                             │
                  ┌──────────┴──────────┐
                  │                     │
                  ▼                     ▼
          ┌────────────────┐    ┌────────────────┐
          │   MATCHED      │    │   UNKNOWN      │
          │   (90 tix)     │    │   (30 tix)     │
          └────────┬───────┘    └────────┬───────┘
                   │                     │
                   │            ┌────────▼─────────┐
                   │            │ LLM VALIDATION   │
                   │            │ (Optional)       │
                   │            │                  │
                   │            │ • OpenAI API     │
                   │            │ • GPT-3.5-turbo  │
                   │            │ • Suggestions    │
                   │            └────────┬─────────┘
                   │                     │
                   └──────────┬──────────┘
                              │
                              ▼
                   ┌────────────────────────────┐
                   │   SEGREGATION LOGIC        │
                   │                            │
                   │ Group by category:         │
                   │ • 100 category sheets      │
                   │ • 1 Unknown_101 sheet      │
                   │ • 1 Summary sheet          │
                   └────────┬───────────────────┘
                            │
                            ▼
                   ┌────────────────────────────┐
                   │   OUTPUT GENERATION        │
                   │   (openpyxl)               │
                   │                            │
                   │ Create multi-sheet Excel:  │
                   │ • 102 worksheets           │
                   │ • Full data preservation   │
                   │ • Statistics included      │
                   └────────┬───────────────────┘
                            │
                            ▼
                 ┌──────────────────────────────┐
                 │ segregated_tickets_output.xlsx│
                 │ (25.73 KB)                    │
                 │                               │
                 │ • System Downtime: 2 tix      │
                 │ • Hardware Failures: 3 tix    │
                 │ • Pharmacy Operations: 5 tix  │
                 │ • ...and 97 more sheets       │
                 │ • Unknown_101: 30 tix         │
                 │ • Summary: Statistics         │
                 └──────────────────────────────┘
```

## Data Flow Diagram

```
TICKET MATCHING FLOW
───────────────────

┌─────────────────────────────────────────────────────────────────┐
│ INPUT TICKET: "Pharmacy system down"                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────┐
        │ CLEANED: "pharmacy system down"     │
        └─────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────┐
        │ VECTORIZED: [0.23, 0.45, 0.12, ...]│
        └─────────────────────────────────────┘
                              │
                ┌─────────────┼─────────────┐
                ▼             ▼             ▼
        ┌─────────────┐  ┌──────────────┐  ┌────────────────┐
        │ System      │  │ Network      │  │ Pharmacy Ops   │
        │ Downtime    │  │ Issues       │  │                │
        │ SIM: 0.85   │  │ SIM: 0.35    │  │ SIM: 0.32      │
        └─────────────┘  └──────────────┘  └────────────────┘
                              │ (highest)
                              ▼
        ┌─────────────────────────────────────┐
        │ CONFIDENCE SCORE: 0.85              │
        │ ≥ 0.3 threshold? YES ✓              │
        └─────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────┐
        │ RESULT: System Downtime category     │
        │ Ticket → System Downtime sheet       │
        └─────────────────────────────────────┘


UNKNOWN TICKET FLOW
───────────────────

┌─────────────────────────────────────────────────────────────────┐
│ INPUT TICKET: "XYZ facility issue"                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    (Standard matching...)
                              │
                              ▼
        ┌─────────────────────────────────────┐
        │ BEST MATCH SCORE: 0.22              │
        │ < 0.3 threshold? YES ✗              │
        └─────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────┐
        │ UNKNOWN TICKET                      │
        │ → Unknown_101 sheet                 │
        └─────────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
                    ▼                   ▼
        ┌─────────────────────┐  ┌──────────────────┐
        │ Manual Review       │  │ LLM Validation   │
        │ Required            │  │ (Optional)       │
        │                     │  │                  │
        │ • Check manually    │  │ • Send to GPT    │
        │ • Reclassify       │  │ • Get suggestion │
        │ • Update sheet     │  │ • Update result  │
        └─────────────────────┘  └──────────────────┘
```

## Module Dependencies

```
ticket_segregation.py (MAIN)
  │
  ├── imports
  │   ├── pandas (Excel I/O)
  │   ├── sklearn.feature_extraction (TF-IDF)
  │   ├── sklearn.metrics.pairwise (Cosine similarity)
  │   └── openai (Optional - LLM)
  │
  └── classes
      └── TicketSegregationEngine
          │
          ├── __init__(tickets_file, categories_file)
          │
          ├── extract_keywords(text)
          │   └── Cleans and tokenizes text
          │
          ├── match_ticket_to_category(description)
          │   ├── Creates TF-IDF vectorizer
          │   ├── Calculates similarities
          │   └── Returns best match + score
          │
          ├── segregate_tickets(confidence_threshold)
          │   ├── Processes all tickets
          │   ├── Matches to categories
          │   └── Separates matched/unknown
          │
          ├── validate_with_llm(use_llm)
          │   ├── Optional OpenAI integration
          │   ├── Validates unknown tickets
          │   └── Updates records
          │
          ├── save_to_excel(output_file)
          │   ├── Creates ExcelWriter
          │   ├── Writes category sheets
          │   ├── Writes summary sheet
          │   └── Saves file
          │
          └── generate_report()
              └── Prints statistics


workflow_utils.py (UTILITIES)
  │
  └── class WorkflowUtils
      │
      ├── @static backup_output()
      │   └── Creates timestamped backup
      │
      ├── @static validate_input_files()
      │   └── Validates file format/columns
      │
      ├── @static get_unmatched_tickets()
      │   └── Extracts Unknown_101 sheet
      │
      ├── @static export_category_statistics()
      │   └── Gets summary statistics
      │
      ├── @static merge_unknown_categorization()
      │   └── Merges LLM suggestions
      │
      └── @static clean_ticket_descriptions()
          └── Removes PII/sensitive data
```

## Processing Pipeline

```
STAGE 1: INITIALIZATION
├─ Load servicenow_tickets.xlsx (120 rows)
├─ Load ticket_categories.xlsx (100 rows)
└─ Initialize TicketSegregationEngine

STAGE 2: TEXT PREPARATION
├─ Extract descriptions from tickets
├─ Clean special characters
├─ Convert to lowercase
└─ Remove stop words (optional)

STAGE 3: VECTORIZATION
├─ Create TF-IDF vectorizer
├─ Fit on all category names + descriptions
├─ Transform all texts to vectors
└─ Create matrix (120 tickets × 100 categories)

STAGE 4: SIMILARITY CALCULATION
└─ For each of 120 tickets:
   ├─ Calculate cosine similarity with 100 categories
   ├─ Find maximum similarity score
   ├─ Identify best matching category
   └─ Store result + confidence score

STAGE 5: SEGREGATION DECISION
└─ For each ticket:
   ├─ IF confidence ≥ 0.3:
   │  └─ Add to category sheet
   └─ ELSE:
      └─ Add to Unknown_101 sheet

STAGE 6: OPTIONAL LLM VALIDATION
├─ IF use_llm = true:
│  └─ For each unknown ticket:
│     ├─ Send to OpenAI API
│     ├─ Get category suggestion
│     └─ Update record with suggestion
└─ Else: Skip LLM

STAGE 7: OUTPUT GENERATION
├─ Create Excel workbook
├─ Write 100 category sheets
├─ Write Unknown_101 sheet
├─ Write Summary sheet
└─ Save as segregated_tickets_output.xlsx

STAGE 8: REPORTING
├─ Calculate statistics
├─ Print summary to console
└─ Display top categories
```

## Configuration Schema

```
.env File Structure
───────────────────

OPENAI_API_KEY=sk-...
│ Purpose: Authentication for OpenAI API
│ Format: String (API key from https://openai.com)
│ Required: Only if USE_LLM_VALIDATION=true
│ Default: None

CONFIDENCE_THRESHOLD=0.3
│ Purpose: Minimum confidence score for categorization
│ Format: Float (0.0 to 1.0)
│ Range:
│   0.0-0.2: Very loose (many matches, low accuracy)
│   0.2-0.4: Loose (balanced)
│   0.4-0.6: Medium (stricter)
│   0.6-1.0: Strict (few matches, high confidence)
│ Default: 0.3
│ Recommended: 0.25-0.35

USE_LLM_VALIDATION=false
│ Purpose: Enable/disable OpenAI GPT validation
│ Format: Boolean (true/false)
│ Default: false
│ Cost: ~$0.001-0.01 per 100 unknown tickets

LLM_MODEL=gpt-3.5-turbo
│ Purpose: Which OpenAI model to use
│ Options: gpt-3.5-turbo, gpt-4
│ Default: gpt-3.5-turbo
│ Note: gpt-4 more expensive but better quality

LLM_MAX_TOKENS=100
│ Purpose: Maximum response length from LLM
│ Format: Integer
│ Default: 100
│ Range: 10-4096

LLM_TEMPERATURE=0.3
│ Purpose: Response randomness/creativity
│ Format: Float (0.0-1.0)
│ 0.0: Deterministic (same answer every time)
│ 0.5: Balanced
│ 1.0: Creative/random
│ Default: 0.3 (lower = more focused)
```

---

**Created**: November 28, 2025  
**Status**: Complete  
**Version**: 1.0
