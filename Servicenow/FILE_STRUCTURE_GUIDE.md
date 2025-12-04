# SmartOps - File-by-File Breakdown

## PROJECT STRUCTURE OVERVIEW

```
smartops-dev/
├── ROOT FILES (Data & Config)
│   ├── README.md
│   ├── requirements.txt
│   ├── .gitignore
│   ├── LLM_USAGE_SUMMARY.md
│   ├── WORKFLOW_DIAGRAMS.md
│   └── DATA FILES (CSV & Excel)
│       ├── incidents_store_pharma_v3.csv
│       ├── kb_store_pharma_v3.csv
│       ├── problems_store_pharma_v3.csv
│       ├── sample_test.csv
│       └── *.xlsx (output files)
│
├── servicenow_api_automation/
│   ├── MAIN ENTRY POINT
│   │   └── main.py
│   │
│   ├── CORE MODULES
│   │   ├── servicenow_client.py (API communication)
│   │   ├── workflow.py (orchestration)
│   │   ├── ticket_segregation_engine.py (TF-IDF matching)
│   │   └── task_assignment.py (team assignment)
│   │
│   ├── MOCK & TEST DATA
│   │   ├── mock_servicenow_data.py (generates fake data)
│   │   ├── mock_tickets.json
│   │   └── mock_categories.json
│   │
│   ├── CONFIGURATION
│   │   ├── config.example.py
│   │   └── requirements.txt
│   │
│   ├── DOCUMENTATION
│   │   ├── README.md
│   │   ├── FLOW_EXPLANATION.md
│   │   └── DELIVERY_SUMMARY.md
│   │
│   ├── UI INTERFACE
│   │   └── streamlit_app.py
│   │
│   └── TESTS
│       └── tests/
│           ├── test_servicenow_client.py
│           ├── test_assignment.py
│           └── test_segregation.py
│
└── ticket_segregation_workflow/
    ├── MAIN ENTRY POINT
    │   └── ticket_segregation.py (standalone segregation)
    │
    ├── UTILITIES
    │   ├── workflow_utils.py (helper functions)
    │   ├── apply_manual_labels.py (label management)
    │   └── generate_sample_data.py (test data generation)
    │
    ├── CONFIGURATION
    │   ├── .env.example
    │   └── requirements.txt
    │
    ├── DOCUMENTATION
    │   ├── README.md
    │   ├── ARCHITECTURE.md
    │   ├── PROJECT_SUMMARY.md
    │   ├── QUICKSTART.md
    │   └── DELIVERY_SUMMARY.txt
    │
    ├── DOCKER
    │   └── Dockerfile
    │
    ├── DATA FILES
    │   ├── ticket_categories.xlsx
    │   ├── segregated_tickets_output.xlsx
    │   └── servicenow_tickets.xlsx
    │
    ├── DEPLOYMENT
    │   ├── git_push.ps1
    │   └── .github/ (CI/CD workflows)
    │
    ├── UI INTERFACE
    │   └── streamlit_app.py
    │
    └── TESTS
        └── tests/
            ├── test_matching.py
            └── test_workflow_utils.py

├── NOTEBOOKS (Exploratory & Production)
│   ├── pharmacy_phi_redaction_production.ipynb (PII masking)
│   ├── pharma_data_processing scrubbed.ipynb (data prep)
│   └── pii_masking.ipynb (legacy)

└── EMBEDDINGS & PICKLES
    ├── KA_embeddings.pkl (knowledge article embeddings)
    └── Problem_embeddings.pkl (problem embeddings)
```

---

## 1. ROOT LEVEL FILES

### README.md
**Purpose:** Project overview and quick start guide
- High-level description of SmartOps
- Key features & benefits
- Quick start instructions
- Contact information

### requirements.txt
**Purpose:** Project-wide Python dependencies
```
pandas
scikit-learn
python-dotenv
openai
requests
openpyxl
streamlit
spacy
presidio-analyzer
presidio-anonymizer
transformers
torch
```

### .gitignore
**Purpose:** Git exclusions
- `__pycache__/`
- `.env` (secrets)
- `*.xlsx` (large outputs)
- `.venv/` (virtual environment)
- `*.pkl` (embeddings)

### LLM_USAGE_SUMMARY.md
**Purpose:** Comprehensive LLM and AI model documentation
- TF-IDF vectorization details
- OpenAI GPT-3.5 integration guide
- spaCy NER setup
- Presidio framework
- HuggingFace biomedical NER
- Cost analysis & recommendations

### WORKFLOW_DIAGRAMS.md
**Purpose:** ASCII diagrams for all 7 phases
- Main workflow overview
- Phase-by-phase flows
- Decision trees
- Error handling
- Execution modes

---

## 2. SERVICENOW API AUTOMATION PROJECT

### 2.1 MAIN ENTRY POINT

#### `main.py` (194 lines)
**Purpose:** Primary orchestration script for the entire workflow
**Key Functions:**
- `load_config()` → Loads credentials from config.py or .env
- `run_with_api()` → Executes with real ServiceNow API
- `run_with_mock()` → Executes with generated mock data
- `run_with_dry_run()` → Simulates without updates

**Usage:**
```bash
python main.py                      # Live mode (updates ServiceNow)
python main.py --dry-run            # Simulation mode
python main.py --mock               # Test mode (no API calls)
python main.py --tickets 200        # Process 200 tickets
```

**Workflow:**
```
main.py
    │
    ├─ Load config & environment variables
    ├─ Parse command-line arguments (--mock, --dry-run)
    ├─ Initialize ServiceNowClient
    ├─ Initialize TicketSegregationWorkflow
    ├─ Run workflow (fetch → segregate → assign → update)
    ├─ Export results to Excel
    └─ Print summary & statistics
```

---

### 2.2 CORE MODULES

#### `servicenow_client.py` (275 lines)
**Purpose:** API client for all ServiceNow REST API calls
**Key Classes:**
```python
class ServiceNowClient:
    - __init__(instance_url, username, password)
    - test_connection()          # Verify API access
    - fetch_tickets(limit, query)     # GET /api/now/table/incident
    - fetch_categories(limit)         # GET /api/now/table/incident_category
    - assign_ticket(number, assigned_to, assignment_group)  # PATCH
    - create_change_log_entry()       # POST audit trail
```

**Key Features:**
- ✅ Authentication (basic auth + session)
- ✅ Pagination (sysparm_offset, sysparm_limit)
- ✅ Rate limiting (backoff & retry)
- ✅ Error handling (retries on timeout)
- ✅ Logging (audit trail)

**Example:**
```python
from servicenow_client import ServiceNowClient

client = ServiceNowClient(
    instance_url="https://dev12345.service-now.com",
    username="your_user",
    password="your_pass"
)

# Test connection
if client.test_connection():
    print("✓ Connected")

# Fetch tickets
tickets = client.fetch_tickets(limit=100, query={'state': '1'})

# Assign ticket
client.assign_ticket(
    ticket_number='INC0010001',
    assigned_to='user@company.com',
    assignment_group='DBA_Group'
)
```

---

#### `workflow.py` (TBD - Orchestration)
**Purpose:** Main orchestrator that ties all modules together
**Pseudo-Structure:**
```python
class TicketSegregationWorkflow:
    def __init__(servicenow_config, team_mapping, confidence_threshold)
    def run(ticket_limit, category_limit, dry_run)
        ├─ fetch_tickets() → servicenow_client.fetch_tickets()
        ├─ fetch_categories() → servicenow_client.fetch_categories()
        ├─ segregate() → TicketSegregationEngine.segregate_tickets()
        ├─ assign_teams() → TaskAssignmentEngine.bulk_assign()
        ├─ update_servicenow() → servicenow_client.assign_ticket() [if not dry_run]
        └─ export_results() → pd.ExcelWriter()
```

---

#### `ticket_segregation_engine.py` (TBD - TF-IDF Matching)
**Purpose:** Core AI/ML matching engine using TF-IDF
**Pseudo-Structure:**
```python
class TicketSegregationEngine:
    def __init__(tickets_df, categories_df)
    
    def match_ticket_to_category(ticket_description)
        ├─ TfidfVectorizer.fit_transform() (100 features, bigrams)
        ├─ cosine_similarity() (ticket vs all categories)
        └─ return (best_category, confidence_score)
    
    def segregate_tickets(confidence_threshold=0.3)
        └─ Loop 100 tickets → match → store in segregated_data[category]
    
    def validate_with_llm(use_llm=True)  [Optional]
        └─ For each unknown ticket → Call GPT-3.5 → Suggest category
    
    def save_to_excel(output_file)
        └─ Create sheets per category + Unknown sheet
```

**TF-IDF Algorithm:**
```
Input: Ticket description ("Database down")
       + 15 category descriptions

Step 1: Vectorization
  └─ TfidfVectorizer converts all text to 100D vectors

Step 2: Similarity
  └─ cosine_similarity(ticket_vector, category_vectors) → [0.05, 0.85, 0.12, ...]

Step 3: Decision
  └─ Max confidence = 0.85
  └─ 0.85 >= 0.3 threshold? YES → MATCHED to best_category
  └─ Store in segregated_data["DB_Admin"]
```

---

#### `task_assignment.py` (TBD - Team Mapping)
**Purpose:** Assign tickets to teams based on TEAM_MAPPING
**Pseudo-Structure:**
```python
class TaskAssignmentEngine:
    def __init__(team_mapping)
    
    def bulk_assign(segregation_results)
        └─ For each matched ticket:
           ├─ Lookup category in TEAM_MAPPING
           ├─ Extract: assigned_to, assignment_group, priority
           └─ Attach to ticket record
    
    def validate_mapping()
        └─ Check all categories have team mappings
```

**TEAM_MAPPING Example:**
```python
TEAM_MAPPING = {
    "DB Admin": {
        "team": "DBA Team",
        "assigned_to": "dba_lead@company.com",
        "assignment_group": "DBA_Group",
        "priority": "critical"
    },
    "Network": {
        "team": "Network Team",
        "assigned_to": "network@company.com",
        "assignment_group": "NET_Group",
        "priority": "high"
    },
    # ... 13 more categories
}
```

---

### 2.3 MOCK & TEST DATA

#### `mock_servicenow_data.py` (Generator)
**Purpose:** Generates fake ServiceNow data for testing
**Key Class:**
```python
class MockServiceNowGenerator:
    def generate_tickets(count=80)      # Create fake tickets
    def generate_categories(count=15)   # Create fake categories
```

**Usage:**
```python
from mock_servicenow_data import MockServiceNowGenerator

gen = MockServiceNowGenerator()
fake_tickets = gen.generate_tickets(100)
fake_categories = gen.generate_categories(15)
```

#### `mock_tickets.json`
**Purpose:** Sample ticket data in JSON format
**Structure:**
```json
[
  {
    "number": "INC0010001",
    "short_description": "Database down",
    "description": "Production database unreachable for 30 minutes",
    "state": "1",
    "created": "2025-12-02T10:00:00Z"
  },
  ...
]
```

#### `mock_categories.json`
**Purpose:** Sample category data
**Structure:**
```json
[
  {
    "sys_id": "cat_001",
    "name": "Database Backup",
    "description": "Database backup and restore issues"
  },
  ...
]
```

---

### 2.4 CONFIGURATION

#### `config.example.py`
**Purpose:** Template for configuration (copy to `config.py`)
**Contents:**
```python
# ServiceNow credentials
SERVICENOW_INSTANCE_URL = "https://dev12345.service-now.com"
SERVICENOW_USERNAME = "your_username"
SERVICENOW_PASSWORD = "your_password"

# API limits
TICKET_LIMIT = 100
CATEGORY_LIMIT = 100

# Confidence threshold for TF-IDF matching
CONFIDENCE_THRESHOLD = 0.3

# Team mapping (categories → team assignments)
TEAM_MAPPING = {
    "DB Admin": {
        "team": "DBA Team",
        "assigned_to": "dba_lead@company.com",
        "assignment_group": "DBA_Group",
        "priority": "critical"
    },
    # ... 14 more mappings
}
```

---

### 2.5 DOCUMENTATION

#### `README.md` (Project-specific)
**Purpose:** ServiceNow API Automation project guide
- Quick start: `python main.py`
- Configuration steps
- API flow explanation
- Troubleshooting

#### `FLOW_EXPLANATION.md` (Detailed diagrams)
**Purpose:** Complete step-by-step flow with visuals
- High-level workflow
- Detailed phase flows
- API request/response examples
- Data transformation pipeline

#### `DELIVERY_SUMMARY.md`
**Purpose:** Project delivery notes & status
- Completed features
- Known limitations
- Next steps
- Performance metrics

---

### 2.6 UI INTERFACE

#### `streamlit_app.py`
**Purpose:** Web dashboard for ticket segregation
**Features:**
```python
# UI Components
- Input: Upload Excel or connect to ServiceNow
- Configuration: Set threshold, team mapping
- Execution: Run segregation
- Output: Display results, download Excel
- Monitoring: Show progress, stats, errors
```

**Usage:**
```bash
streamlit run streamlit_app.py
# Opens: http://localhost:8501
```

---

### 2.7 TESTS

#### `tests/test_servicenow_client.py`
**Purpose:** Unit tests for ServiceNow API client
**Test Cases:**
```python
def test_connection()           # Verify API connectivity
def test_fetch_tickets()        # Mock API responses
def test_fetch_categories()     # Category fetching
def test_rate_limiting()        # Rate limit handling
def test_error_handling()       # Error recovery
```

#### `tests/test_assignment.py`
**Purpose:** Tests for team assignment logic
```python
def test_team_mapping()         # Verify mappings
def test_bulk_assign()          # Assign multiple tickets
def test_missing_mapping()      # Handle missing teams
```

#### `tests/test_segregation.py`
**Purpose:** Tests for TF-IDF matching
```python
def test_tfidf_matching()       # Confidence scoring
def test_unknown_tickets()      # Below-threshold handling
def test_llm_validation()       # GPT-3.5 integration (mock)
```

---

## 3. TICKET SEGREGATION WORKFLOW PROJECT

### 3.1 MAIN ENTRY POINT

#### `ticket_segregation.py` (310 lines - Standalone)
**Purpose:** Standalone ticket segregation without ServiceNow API
**Key Class:**
```python
class TicketSegregationEngine:
    def __init__(tickets_file, categories_file, tickets_df, categories_df)
    def match_ticket_to_category(ticket_description)
    def segregate_tickets(confidence_threshold=0.3)
    def validate_with_llm(use_llm=False)        # Optional OpenAI
    def save_to_excel(output_file)
    def generate_segregation_report()
```

**Usage:**
```python
from ticket_segregation import TicketSegregationEngine

# From files
engine = TicketSegregationEngine(
    tickets_file='servicenow_tickets.xlsx',
    categories_file='ticket_categories.xlsx'
)

# Or from DataFrames (from API)
engine = TicketSegregationEngine(
    tickets_df=tickets_df,
    categories_df=categories_df
)

# Segregate
engine.segregate_tickets(confidence_threshold=0.3)

# Optional: Validate with GPT-3.5
engine.validate_with_llm(use_llm=True)

# Export
engine.save_to_excel('segregated_tickets_output.xlsx')
```

**Algorithm Details:**
```
For each ticket:
  1. Extract keywords from description
  2. TF-IDF vectorize vs 15 category descriptions
  3. Calculate cosine similarity scores
  4. Find best match (highest score)
  5. Compare vs threshold (0.3):
     ├─ score >= 0.3 → MATCHED (store in category sheet)
     └─ score < 0.3  → UNKNOWN (optional LLM validation)
  6. Output: segregated_data[category_name] = [ticket1, ticket2, ...]

Save to Excel:
  ├─ Sheet: Summary (stats, charts)
  ├─ Sheet: DB Admin (15 matched tickets)
  ├─ Sheet: Network (12 matched tickets)
  ├─ ... (13 more category sheets)
  └─ Sheet: Unknown_101 (25 unmatched tickets)
```

---

### 3.2 UTILITIES

#### `workflow_utils.py` (Helper functions)
**Purpose:** Common utility functions
```python
def load_excel_file(path)          # Read Excel
def save_excel_file(df, path)      # Write Excel
def extract_keywords(text)         # Text preprocessing
def validate_dataframe(df)         # Data validation
def generate_report(results)       # Report generation
```

#### `apply_manual_labels.py` (Label management)
**Purpose:** Manually label tickets for training data
**Features:**
- Interactive CLI for labeling
- Save labels to Excel
- Review labeled data
- Generate training dataset

**Usage:**
```bash
python apply_manual_labels.py
# Interactive: "Enter ticket number, category, confidence"
```

#### `generate_sample_data.py` (Test data)
**Purpose:** Generate sample tickets & categories for testing
```python
def generate_sample_tickets(count=80)
def generate_sample_categories(count=15)
```

---

### 3.3 CONFIGURATION

#### `.env.example`
**Purpose:** Template for environment variables
```
SERVICENOW_INSTANCE_URL=https://dev12345.service-now.com
SERVICENOW_USERNAME=your_username
SERVICENOW_PASSWORD=your_password
OPENAI_API_KEY=sk-xxx...xxx
CONFIDENCE_THRESHOLD=0.3
TICKET_LIMIT=100
```

#### `requirements.txt`
**Purpose:** Project dependencies
```
pandas
scikit-learn
openpyxl
python-dotenv
openai
spacy
```

---

### 3.4 DOCUMENTATION

#### `README.md`
**Purpose:** Quick start & usage
```markdown
# Ticket Segregation Workflow

## Quick Start
```bash
python ticket_segregation.py
```

## Features
- TF-IDF matching
- Optional LLM validation
- Excel export
- Category breakdown
```

#### `ARCHITECTURE.md`
**Purpose:** System design & architecture
- Component diagram
- Data flow
- Algorithm details
- Performance metrics

#### `PROJECT_SUMMARY.md`
**Purpose:** High-level project overview
- Goals & objectives
- Deliverables
- Status
- Metrics

#### `QUICKSTART.md`
**Purpose:** 5-minute setup guide
```
1. Install: pip install -r requirements.txt
2. Copy .env.example to .env
3. Run: python ticket_segregation.py
4. Output: segregated_tickets_output.xlsx
```

#### `DELIVERY_SUMMARY.txt`
**Purpose:** Release notes & deliverables
- What was built
- Features completed
- Known issues
- Next phase

---

### 3.5 DOCKER

#### `Dockerfile`
**Purpose:** Containerization for deployment
```dockerfile
FROM python:3.11
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "ticket_segregation.py"]
```

**Usage:**
```bash
docker build -t smartops-seg .
docker run smartops-seg
```

---

### 3.6 DATA FILES

#### `ticket_categories.xlsx`
**Purpose:** Known categories (master data)
**Columns:**
```
Category_ID | Category_Name | Description
    cat_001 | DB Admin      | Database administration issues
    cat_002 | Network       | Network connectivity issues
    ...
```

#### `segregated_tickets_output.xlsx`
**Purpose:** Output file from segregation
**Sheets:**
```
Summary          → Statistics & charts
DB Admin (15)    → Matched tickets
Network (12)     → Matched tickets
...
Unknown_101 (25) → Unmatched tickets (need manual review)
```

#### `servicenow_tickets.xlsx`
**Purpose:** Sample input file
**Columns:**
```
Ticket_Number | Short_Description | Description | Created
   INC0010001 | Database down     | Production DB...
   INC0010002 | VPN issue         | Cannot connect...
```

---

### 3.7 DEPLOYMENT

#### `git_push.ps1` (PowerShell script)
**Purpose:** Automated Git commit & push
```powershell
# Stages changes, commits with message, pushes to remote
.\git_push.ps1
```

#### `.github/` (CI/CD workflows)
**Purpose:** GitHub Actions automation
- Auto-test on push
- Auto-deploy to staging
- Auto-release to production

---

### 3.8 TESTS

#### `tests/test_matching.py`
**Purpose:** Unit tests for TF-IDF matching
```python
def test_exact_match()         # Perfect match scoring
def test_partial_match()       # Partial match handling
def test_no_match()            # Below-threshold handling
def test_tfidf_vectorization() # Vector accuracy
```

#### `tests/test_workflow_utils.py`
**Purpose:** Tests for utility functions
```python
def test_load_excel()          # File I/O
def test_extract_keywords()    # Text preprocessing
def test_generate_report()     # Report generation
```

---

## 4. JUPYTER NOTEBOOKS (Exploratory)

### `pharmacy_phi_redaction_production.ipynb` (Production-ready)
**Purpose:** PHI/PII redaction for pharmacy records
**Key Components:**
1. **Presidio Framework** - PII detection
2. **spaCy NER** - Named entity recognition (11MB small model)
3. **HuggingFace Biomedical NER** - Disease detection (107 entity types)
4. **Custom Recognizers** - Rx#, Fill ID, MRN patterns
5. **Anonymization Engine** - Redact sensitive data

**Performance:**
- ~200ms per record
- 1000+ records per hour
- HIPAA-compliant

### `pharma_data_processing scrubbed.ipynb`
**Purpose:** Data preparation & cleaning
- Load CSV files
- Clean & normalize
- Handle missing values
- Export for downstream processing

### `pii_masking.ipynb` (Legacy)
**Purpose:** Earlier version of PHI redaction
- Kept for reference
- Superseded by `pharmacy_phi_redaction_production.ipynb`

---

## 5. EMBEDDINGS & PICKLES

### `KA_embeddings.pkl`
**Purpose:** Pre-computed embeddings for knowledge articles
**Format:** Pickled NumPy arrays
**Size:** ~5MB
**Usage:** Knowledge article similarity matching

### `Problem_embeddings.pkl`
**Purpose:** Pre-computed embeddings for problems
**Format:** Pickled NumPy arrays
**Size:** ~3MB
**Usage:** Problem pattern matching

---

## QUICK REFERENCE: FILE DEPENDENCIES

```
main.py (servicenow_api_automation)
  ├─ imports servicenow_client.py
  ├─ imports workflow.py
  ├─ imports task_assignment.py
  ├─ imports config.py
  └─ imports mock_servicenow_data.py

workflow.py
  ├─ imports servicenow_client.py
  ├─ imports ticket_segregation_engine.py
  └─ imports task_assignment.py

ticket_segregation_engine.py
  ├─ imports sklearn (TfidfVectorizer, cosine_similarity)
  ├─ imports pandas (DataFrame operations)
  └─ imports openai (optional, for LLM validation)

ticket_segregation.py (ticket_segregation_workflow)
  ├─ imports sklearn
  ├─ imports pandas
  ├─ imports workflow_utils.py
  └─ imports openai (optional)

streamlit_app.py
  ├─ imports servicenow_client.py
  ├─ imports workflow.py
  └─ imports pandas & streamlit
```

---

## EXECUTION FLOWS

### Flow 1: ServiceNow API Automation (Recommended)
```
python servicenow_api_automation/main.py
    │
    ├─ Load config.py & .env
    ├─ Create ServiceNowClient
    ├─ Create TicketSegregationWorkflow
    ├─ Fetch tickets & categories via API
    ├─ Segregate using TF-IDF
    ├─ Assign teams using TEAM_MAPPING
    ├─ Update ServiceNow (or dry-run)
    └─ Export Excel + print summary
```

### Flow 2: Standalone Segregation (Files Only)
```
python ticket_segregation_workflow/ticket_segregation.py
    │
    ├─ Load tickets.xlsx
    ├─ Load categories.xlsx
    ├─ Segregate using TF-IDF
    ├─ Optional: LLM validation
    └─ Export Excel
```

### Flow 3: Web Dashboard
```
streamlit run servicenow_api_automation/streamlit_app.py
    │
    ├─ Upload files or API config
    ├─ Configure parameters
    ├─ Execute workflow
    └─ Download results
```

---

## SUMMARY TABLE

| Component | Files | Purpose | Technology |
|-----------|-------|---------|-----------|
| **API Client** | servicenow_client.py | ServiceNow REST API | requests, auth |
| **Matching Engine** | ticket_segregation_engine.py | TF-IDF + cosine sim | scikit-learn |
| **Workflow** | workflow.py, main.py | Orchestration | Python async |
| **Team Assignment** | task_assignment.py | Route to teams | config TEAM_MAPPING |
| **Optional LLM** | (integrated) | GPT-3.5 validation | OpenAI API |
| **Excel Export** | (all save methods) | Output format | openpyxl |
| **Web UI** | streamlit_app.py | Dashboard | streamlit |
| **PII Redaction** | pharmacy_*.ipynb | Data masking | Presidio, spaCy |
| **Testing** | tests/*.py | Validation | pytest |

---

## KEY TAKEAWAYS

1. **Two entry points:**
   - `servicenow_api_automation/main.py` → Real-time with API
   - `ticket_segregation_workflow/ticket_segregation.py` → File-based standalone

2. **Core algorithm:** TF-IDF + Cosine Similarity (fast, free, no API)

3. **Optional enhancement:** GPT-3.5 for unknown tickets validation

4. **Output:** Excel with per-category sheets + statistics

5. **Safe testing:** `--dry-run` mode shows changes without updating ServiceNow

6. **Bonus:** PHI redaction notebooks for sensitive data handling

7. **Configuration:** Copy `config.example.py` and `.env.example` to customize

---

This file-wise breakdown shows how all components work together to automate ticket segregation and team assignment! 🚀
