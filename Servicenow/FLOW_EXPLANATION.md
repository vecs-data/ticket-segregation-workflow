# ServiceNow API Automation - Complete Flow Explanation

## 1. HIGH-LEVEL WORKFLOW DIAGRAM

```
┌────────────────────────────────────────────────────────────────────┐
│                   SERVICENOW API AUTOMATION FLOW                   │
└────────────────────────────────────────────────────────────────────┘

                            START
                              │
                              ▼
                    ┌─────────────────┐
                    │  Load Config    │
                    │ (credentials,   │
                    │  team mapping)  │
                    └────────┬────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │  Test ServiceNow Connection  │
              │  (verify API access)         │
              └──────────────┬───────────────┘
                             │
                    ┌────────▼────────┐
                    │  Connection OK? │
                    └────────┬────────┘
                             │ YES
                             ▼
              ┌──────────────────────────────┐
              │  FETCH DATA FROM SERVICENOW  │
              │  ├─ Get Open Tickets         │
              │  └─ Get Categories           │
              └──────────────┬───────────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │  SEGREGATE TICKETS           │
              │  ├─ TF-IDF Vectorization     │
              │  ├─ Cosine Similarity Match  │
              │  ├─ Calculate Confidence     │
              │  └─ Apply Threshold (0.3)    │
              └──────────────┬───────────────┘
                             │
                ┌────────────▼────────────┐
                │  Matched? (Confidence   │
                │  >= 0.3)                │
                └────┬──────────────┬─────┘
                 YES │              │ NO
            ┌────────▼──────┐   ┌──▼───────────┐
            │  ASSIGN TO    │   │  UNKNOWN     │
            │  CATEGORY     │   │  CATEGORY    │
            │  TEAM         │   │  (Manual      │
            │  (Automated)  │   │   Review)    │
            └────────┬──────┘   └──┬───────────┘
                     │              │
                     └──────┬───────┘
                            ▼
              ┌──────────────────────────────┐
              │  UPDATE SERVICENOW           │
              │  ├─ Assign Ticket to User    │
              │  ├─ Set Assignment Group     │
              │  ├─ Create Change Log Entry  │
              │  └─ Update Status            │
              └──────────────┬───────────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │  GENERATE REPORT             │
              │  ├─ Total Processed          │
              │  ├─ Matched vs Unknown       │
              │  ├─ Assignment Summary       │
              │  └─ Export Excel             │
              └──────────────┬───────────────┘
                             │
                             ▼
                            END

```

---

## 2. DETAILED STEP-BY-STEP FLOW

### STEP 1: Configuration & Authentication

```
User Input
    │
    ▼
┌─────────────────────────────────┐
│ Load Configuration              │
├─────────────────────────────────┤
│ • SERVICENOW_INSTANCE_URL       │
│ • SERVICENOW_USERNAME           │
│ • SERVICENOW_PASSWORD (or PAT)  │
│ • CONFIDENCE_THRESHOLD (0.3)    │
│ • TEAM_MAPPING (categories→teams)
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│ Create ServiceNow Client        │
├─────────────────────────────────┤
│ client = ServiceNowClient(      │
│   instance_url,                 │
│   username,                     │
│   password                      │
│ )                               │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│ Test Connection                 │
├─────────────────────────────────┤
│ HTTP GET /api/now/table/sys_user│
│ Expected: 200 OK                │
└──────────┬──────────────────────┘
           │
      ┌────▼────┐
      │ Success?│
      └────┬────┘
           │ YES
           ▼ (Continue)

```

**Code Example:**
```python
from servicenow_client import ServiceNowClient

client = ServiceNowClient(
    instance_url="https://dev12345.service-now.com",
    username="your_username",
    password="your_password"
)

# Test connection
if client.test_connection():
    print("✓ Connected to ServiceNow")
else:
    print("✗ Connection failed")
```

---

### STEP 2: Fetch Tickets from ServiceNow API

```
┌────────────────────────────────────────┐
│ FETCH TICKETS (with Pagination)        │
└────────────────────────────────────────┘
           │
           ▼
┌────────────────────────────────────────┐
│ HTTP Request #1                        │
├────────────────────────────────────────┤
│ GET /api/now/table/incident            │
│ Parameters:                            │
│  • sysparm_limit=100                   │
│  • sysparm_offset=0                    │
│  • sysparm_query=state=1 (open tickets)│
└──────────┬───────────────────────────┘
           │
           ▼
┌────────────────────────────────────────┐
│ Response: 100 tickets                  │
├────────────────────────────────────────┤
│ [                                      │
│   {                                    │
│     "number": "INC0010001",            │
│     "short_description": "...",        │
│     "description": "...",              │
│     "state": "1"                       │
│   },                                   │
│   ... (99 more)                        │
│ ]                                      │
└──────────┬───────────────────────────┘
           │
    ┌──────▼──────┐
    │ More tickets?
    └──────┬──────┘
         YES │
    ┌────────▼────────┐
    │ HTTP Request #2 │ (offset=100, limit=100)
    │ ... iterate ...  │
    └──────────────────┘
    
    NO │
    ▼
Store all tickets in memory
(Default: 100 limit)

```

**Code Example:**
```python
# Fetch tickets
tickets = client.fetch_tickets(
    limit=100,
    query={'state': '1'},  # Open tickets only
    fields=['number', 'short_description', 'description', 'category']
)

# Result: List of 100 tickets with metadata
# [
#   {'number': 'INC0010001', 'short_description': 'Database down', ...},
#   {'number': 'INC0010002', 'short_description': 'VPN issue', ...},
#   ...
# ]
```

---

### STEP 3: Fetch Categories from ServiceNow API

```
┌────────────────────────────────────────┐
│ FETCH CATEGORIES (from incident table) │
└────────────────────────────────────────┘
           │
           ▼
┌────────────────────────────────────────┐
│ HTTP Request                           │
├────────────────────────────────────────┤
│ GET /api/now/table/incident_category   │
│ Parameters:                            │
│  • sysparm_limit=100                   │
│  • sysparm_offset=0                    │
└──────────┬───────────────────────────┘
           │
           ▼
┌────────────────────────────────────────┐
│ Response: Categories                   │
├────────────────────────────────────────┤
│ [                                      │
│   {                                    │
│     "sys_id": "cat_12345",             │
│     "name": "Database Backup",         │
│     "description": "Database backup..." │
│   },                                   │
│   {                                    │
│     "sys_id": "cat_67890",             │
│     "name": "VPN Connectivity",        │
│     "description": "VPN issues..."     │
│   },                                   │
│   ... (98 more)                        │
│ ]                                      │
└──────────┬───────────────────────────┘
           │
           ▼
Store all categories (100 categories)

```

**Code Example:**
```python
# Fetch categories
categories = client.fetch_categories(
    limit=100,
    fields=['sys_id', 'name', 'description']
)

# Result: List of 100 categories
# [
#   {'sys_id': 'cat_001', 'name': 'Database Backup', 'description': '...'},
#   {'sys_id': 'cat_002', 'name': 'VPN Connectivity', 'description': '...'},
#   ...
# ]
```

---

### STEP 4: Initialize Segregation Engine

```
┌─────────────────────────────────────┐
│ Convert API Response to DataFrames  │
└─────────────────────────────────────┘
           │
           ▼
    ┌──────────────────┐
    │ Tickets DataFrame │
    ├──────────────────┤
    │ Columns:         │
    │ • Ticket_Number  │
    │ • Description    │
    │ • Short_Desc...  │
    └────────┬─────────┘
             │
    ┌────────▼────────┐
    │ Categories DF   │
    ├─────────────────┤
    │ Columns:        │
    │ • Category_Name │
    │ • Description   │
    └────────┬────────┘
             │
             ▼
┌──────────────────────────────────────┐
│ TicketSegregationEngine              │
├──────────────────────────────────────┤
│ engine = TicketSegregationEngine(    │
│   tickets_df=tickets_df,             │
│   categories_df=categories_df        │
│ )                                    │
└──────────┬───────────────────────────┘
           │
           ▼
      READY TO MATCH

```

**Code Example:**
```python
import pandas as pd
from ticket_segregation_engine import TicketSegregationEngine

# Convert to DataFrames
tickets_df = pd.DataFrame(tickets)
categories_df = pd.DataFrame(categories)

# Rename columns to match engine expectations
tickets_df.rename(columns={'number': 'Ticket_Number'}, inplace=True)
categories_df.rename(columns={'name': 'Category_Name'}, inplace=True)

# Initialize engine
engine = TicketSegregationEngine(
    tickets_df=tickets_df,
    categories_df=categories_df
)
```

---

### STEP 5: Segregate Tickets Using TF-IDF Matching

```
┌────────────────────────────────────────────┐
│ FOR EACH TICKET (100 iterations)           │
└────────────────────────────────────────────┘
           │
           ▼
    ┌──────────────────┐
    │ Ticket #1        │
    │ Description:     │
    │ "Database down"  │
    └────────┬─────────┘
             │
             ▼
┌────────────────────────────────────────┐
│ STEP 1: TF-IDF VECTORIZATION           │
├────────────────────────────────────────┤
│ Input:                                 │
│  • Ticket text: "database down"        │
│  • All 100 category descriptions       │
│                                        │
│ TfidfVectorizer(                       │
│   max_features=100,                    │
│   ngram_range=(1, 2)  # 1-2 word grams│
│ )                                      │
│                                        │
│ Output:                                │
│  • Vector for ticket: [0.2, 0.5, ...]  │
│  • Vector for category 1: [0.1, 0.3...]│
│  • Vector for category 2: [0.4, 0.2...]│
│  • ... (100 category vectors)          │
└────────┬───────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ STEP 2: COSINE SIMILARITY CALCULATION  │
├────────────────────────────────────────┤
│ Compare ticket vector with each        │
│ category vector:                       │
│                                        │
│ similarity = cos(angle between vectors)│
│                                        │
│ Results:                               │
│  Category 1: 0.05  (5% match)          │
│  Category 2: 0.85  (85% match) ← MAX  │
│  Category 3: 0.12  (12% match)         │
│  ... (98 more)                         │
└────────┬───────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ STEP 3: FIND BEST MATCH                │
├────────────────────────────────────────┤
│ Max confidence: 0.85                   │
│ Category: "Database Backup"            │
│ Ticket: "INC0010001"                   │
└────────┬───────────────────────────────┘
         │
    ┌────▼─────┐
    │Confidence│
    │>= 0.3?   │
    └────┬─────┘
         │
   ┌─────┴─────┐
  YES          NO
   │            │
   ▼            ▼
MATCHED    UNKNOWN
(Category:  (Category:
Database   "Unknown")
Backup)    

┌────────────────────────────┐
│ Store Result:              │
├────────────────────────────┤
│ {                          │
│   'Ticket_Number': 'INC1', │
│   'Category': 'DB Backup', │
│   'Confidence': 0.85,      │
│   'Status': 'Matched'      │
│ }                          │
└────────────────────────────┘

```

**Code Example:**
```python
# Segregate tickets
results = engine.segregate_tickets(
    confidence_threshold=0.3
)

# Result: List of segregation results
# [
#   {
#     'Ticket_Number': 'INC0010001',
#     'Category': 'Database Backup',
#     'Confidence': 0.85,
#     'Status': 'Matched'
#   },
#   {
#     'Ticket_Number': 'INC0010002',
#     'Category': 'Unknown',
#     'Confidence': 0.25,
#     'Status': 'Unknown'
#   },
#   ...
# ]
```

---

### STEP 6: Team Assignment Automation

```
┌────────────────────────────────────┐
│ FOR EACH SEGREGATED TICKET         │
└────────────────────────────────────┘
           │
           ▼
    ┌──────────────────┐
    │ Ticket Result    │
    ├──────────────────┤
    │ Ticket: INC0001  │
    │ Category:        │
    │ "Database Backup"│
    │ Confidence: 0.85 │
    │ Status: Matched  │
    └────────┬─────────┘
             │
             ▼
┌────────────────────────────────────┐
│ LOOKUP TEAM MAPPING                │
├────────────────────────────────────┤
│ TEAM_MAPPING = {                   │
│   "Database Backup": {             │
│     "team": "DBA Team",            │
│     "assigned_to":                 │
│       "dba_lead@company.com",      │
│     "assignment_group":            │
│       "DBA_Group",                 │
│     "priority": "critical"         │
│   },                               │
│   ... (14 more categories)         │
│ }                                  │
└────────┬───────────────────────────┘
         │
    ┌────▼─────────┐
    │Category in   │
    │TEAM_MAPPING? │
    └────┬────┬────┘
      YES│    │NO
         ▼    ▼
      ASSIGN UNKNOWN
      TO TEAM (No Assignment)
      
┌──────────────────────────┐
│ Assignment Result:       │
├──────────────────────────┤
│ {                        │
│   'Assigned_To':         │
│   'dba_lead@company.com',│
│   'Assignment_Group':    │
│   'DBA_Group',           │
│   'Team': 'DBA Team',    │
│   'Priority': 'critical' │
│ }                        │
└──────────────────────────┘

```

**Code Example:**
```python
from task_assignment import TaskAssignmentEngine

team_mapping = {
    "Database Backup": {
        "team": "DBA Team",
        "assigned_to": "dba_lead@company.com",
        "assignment_group": "DBA_Group",
        "priority": "critical"
    },
    # ... 14 more categories
}

assignment_engine = TaskAssignmentEngine(team_mapping)

# Bulk assign all tickets
results = assignment_engine.bulk_assign(segregation_results)

# Result: Each result now has assignment details
# [
#   {
#     'Ticket_Number': 'INC0010001',
#     'Category': 'Database Backup',
#     'Assigned_To': 'dba_lead@company.com',
#     'Assignment_Group': 'DBA_Group',
#     'Team': 'DBA Team',
#     'Priority': 'critical',
#     'Assignment_Status': 'Assigned'
#   },
#   ...
# ]
```

---

### STEP 7: Update ServiceNow

```
┌─────────────────────────────────────┐
│ FOR EACH ASSIGNED TICKET (DRY-RUN?) │
└─────────────────────────────────────┘
           │
    ┌──────▼──────┐
    │ Dry-Run?    │
    └──────┬──────┘
         YES│
           │ ┌─────────────────────────┐
           │ │ LOG: "Would update..."  │
           │ │ (Simulation - no changes)
           │ └─────────────────────────┘
           │
         NO│
           ▼
┌─────────────────────────────────────────┐
│ HTTP PATCH REQUEST (Update Ticket)      │
├─────────────────────────────────────────┤
│ PATCH /api/now/table/incident           │
│                                         │
│ Query:                                  │
│  sysparm_query=number=INC0010001        │
│                                         │
│ Body (JSON):                            │
│ {                                       │
│   "assigned_to": "dba_lead@company.com",│
│   "assignment_group": "DBA_Group"       │
│ }                                       │
│                                         │
│ Response: 200 OK                        │
│ {                                       │
│   "result": {                           │
│     "number": "INC0010001",             │
│     "assigned_to": "dba_lead@...",      │
│     "updated": "2025-12-02T10:30:00Z"   │
│   }                                     │
│ }                                       │
└────────┬────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│ CREATE CHANGE LOG ENTRY (Audit Trail)   │
├─────────────────────────────────────────┤
│ POST /api/now/table/cmdb_change_log     │
│                                         │
│ Body:                                   │
│ {                                       │
│   "ticket_number": "INC0010001",        │
│   "category": "Database Backup",        │
│   "confidence_score": "85%",            │
│   "assigned_to": "dba_lead@company.com",│
│   "change_type": "Automated Segregation"│
│ }                                       │
│                                         │
│ Purpose: Track automation history       │
└────────┬────────────────────────────────┘
         │
         ▼
    ✓ TICKET UPDATED
    ✓ AUDIT LOGGED

```

**Code Example:**
```python
# Update ticket in ServiceNow
if not dry_run:
    # Assign ticket
    client.assign_ticket(
        ticket_number='INC0010001',
        assigned_to='dba_lead@company.com',
        assignment_group='DBA_Group'
    )
    
    # Create audit log
    client.create_change_log_entry(
        ticket_number='INC0010001',
        category='Database Backup',
        confidence=0.85,
        assigned_to='dba_lead@company.com'
    )
else:
    print("[DRY RUN] Would assign INC0010001 to dba_lead@company.com")
```

---

### STEP 8: Generate Report & Export

```
┌──────────────────────────────────────┐
│ AGGREGATE RESULTS                    │
├──────────────────────────────────────┤
│ • Total Tickets: 100                 │
│ • Matched: 75 (75%)                  │
│ • Unknown: 25 (25%)                  │
│ • Assigned: 75 tickets               │
│ • Failed: 0 tickets                  │
└────────┬─────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│ EXPORT TO EXCEL                      │
├──────────────────────────────────────┤
│ segregated_tickets_output.xlsx        │
│                                      │
│ Sheets:                              │
│ • Summary                            │
│   - Total: 100                       │
│   - Matched: 75                      │
│   - Unknown: 25                      │
│                                      │
│ • Database Backup (15 tickets)       │
│   - INC0010001, INC0010005, ...      │
│                                      │
│ • VPN Connectivity (12 tickets)      │
│   - INC0010002, INC0010008, ...      │
│                                      │
│ • User Access (10 tickets)           │
│   - ...                              │
│                                      │
│ • ... (other categories)             │
│                                      │
│ • Unknown (25 tickets)               │
│   - INC0010003, INC0010004, ...      │
│   - (Manual review needed)           │
└────────┬─────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│ PRINT SUMMARY REPORT                 │
├──────────────────────────────────────┤
│ ✓ Segregation Complete!              │
│   - Loaded 100 tickets               │
│   - Segregated 100 tickets           │
│   - Matched: 75 tickets              │
│   - Unknown: 25 tickets              │
│   - Assigned: 75 tickets             │
│   - Failed: 0 tickets                │
│                                      │
│ Top Categories by Count:             │
│   1. Unknown: 25 tickets             │
│   2. Database Backup: 15 tickets     │
│   3. VPN Connectivity: 12 tickets    │
│   4. User Access: 10 tickets         │
│   ...                                │
└──────────────────────────────────────┘

```

---

## 3. COMPLETE END-TO-END FLOW DIAGRAM

```
┌───────────────────────────────────────────────────────────────────┐
│                                                                   │
│              USER / ADMINISTRATOR                                 │
│                     │                                             │
│                     ▼                                             │
│         ┌─────────────────────┐                                   │
│         │  python main.py     │                                   │
│         │  (or streamlit run) │                                   │
│         └──────────┬──────────┘                                   │
│                    │                                              │
├────────────────────┼──────────────────────────────────────────────┤
│                    │                                              │
│              SMARTOPS PROJECT                                     │
│                    │                                              │
│         ┌──────────▼──────────┐                                   │
│         │  Load Configuration │                                   │
│         │  & Create Client    │                                   │
│         └──────────┬──────────┘                                   │
│                    │                                              │
│         ┌──────────▼──────────────┐                               │
│         │  Test Connection to     │                               │
│         │  ServiceNow API         │                               │
│         └──────────┬──────────────┘                               │
│                    │ SUCCESS                                      │
│         ┌──────────▼──────────────┐                               │
│         │ Fetch 100 Tickets       │                               │
│         │ (Pagination + Rate      │                               │
│         │  Limiting)              │                               │
│         └──────────┬──────────────┘                               │
│                    │                                              │
│         ┌──────────▼──────────────┐                               │
│         │ Fetch 15 Categories     │                               │
│         └──────────┬──────────────┘                               │
│                    │                                              │
│         ┌──────────▼──────────────┐                               │
│         │ Convert to DataFrames   │                               │
│         └──────────┬──────────────┘                               │
│                    │                                              │
│         ┌──────────▼──────────────┐                               │
│         │ TF-IDF Segregation      │                               │
│         │ (100 iterations)        │                               │
│         │ ├─ Vectorize text       │                               │
│         │ ├─ Cosine similarity    │                               │
│         │ ├─ Find best match      │                               │
│         │ └─ Check threshold      │                               │
│         └──────────┬──────────────┘                               │
│                    │                                              │
│         ┌──────────▼──────────────┐                               │
│         │ Team Assignment         │                               │
│         │ ├─ Lookup team mapping  │                               │
│         │ ├─ Assign to user/group │                               │
│         │ └─ Track assignments    │                               │
│         └──────────┬──────────────┘                               │
│                    │                                              │
│    ┌───────────────▼───────────────┐                              │
│    │       DRY-RUN MODE?            │                             │
│    └───────┬───────────────┬────────┘                             │
│        YES │               │ NO                                   │
│          ┌─▼─┐          ┌──▼─────────┐                            │
│          │LOG│          │ HTTP PATCH  │                           │
│          │   │          │ /api/now/   │                           │
│          │   │          │ table/      │                           │
│          │   │          │ incident    │                           │
│          │   │          │             │                           │
│          │   │          │ + Create    │                           │
│          │   │          │ Change Log  │                           │
│          └──┬┘          └──┬──────────┘                           │
│             │              │                                      │
│             │              ▼                                      │
│             │         ┌──────────────┐                            │
│             │         │ SERVICENOW   │                            │
│             │         │ UPDATED ✓    │                            │
│             │         │ AUDIT LOGGED │                            │
│             │         └──────┬───────┘                            │
│             │                │                                    │
│             └────────┬───────┘                                    │
│                      ▼                                            │
│         ┌────────────────────────┐                                │
│         │ Generate Report        │                                │
│         │ & Export Excel         │                                │
│         └────────────┬───────────┘                                │
│                      │                                            │
│                      ▼                                            │
│         ┌────────────────────────┐                                │
│         │ Display Results        │                                │
│         │ • Total: 100           │                                │
│         │ • Matched: 75          │                                │
│         │ • Unknown: 25          │                                │
│         │ • Assigned: 75         │                                │
│         └────────────┬───────────┘                                │
│                      │                                            │
├──────────────────────┼──────────────────────────────────────────────┤
│                      │                                              │
│              USER SEES OUTPUT                                       │
│                      ▼                                              │
│         ┌────────────────────────┐                                 │
│         │ Excel File             │                                 │
│         │ + Console Report       │                                 │
│         │ + (Optional) Streamlit UI                                │
│         └────────────────────────┘                                 │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

---

## 4. DATA TRANSFORMATION FLOW

```
INPUT (ServiceNow API)
        │
        ├─ Ticket Data
        │  {
        │    "number": "INC0010001",
        │    "short_description": "Database down",
        │    "description": "Production DB unreachable for 30 minutes",
        │    "created": "2025-12-02T10:00:00Z",
        │    "state": "1"
        │  }
        │
        ├─ Category Data
        │  {
        │    "sys_id": "cat_001",
        │    "name": "Database Backup",
        │    "description": "Database backup and restore issues"
        │  }
        │
        ▼
    ┌─────────────────────────┐
    │ Convert to DataFrames   │
    └─────────────┬───────────┘
        │
        ▼
    Tickets DataFrame          Categories DataFrame
    ┌──────────────────┐      ┌──────────────────┐
    │ Ticket_Number   │      │ Category_ID      │
    │ Short_Desc...   │      │ Category_Name    │
    │ Description     │      │ Description      │
    └──────────────────┘      └──────────────────┘
    
        │
        ▼
    ┌─────────────────────────┐
    │ TF-IDF Vectorization    │
    │ (Bag-of-words approach) │
    └─────────────┬───────────┘
        │
        ▼
    Ticket Vector         Category Vectors
    [0.2, 0.5,          [0.1, 0.3, ...] (Cat 1)
     0.1, 0.8, ...]     [0.4, 0.2, ...] (Cat 2)
                        ...
        │
        ▼
    ┌──────────────────────┐
    │ Cosine Similarity    │
    │ Matching             │
    └──────────┬───────────┘
        │
        ▼
    Confidence Scores
    Cat 1: 0.05
    Cat 2: 0.85  ◄─── MAX (Best Match)
    Cat 3: 0.12
    ...
        │
        ▼
    ┌─────────────────────────────┐
    │ Apply Threshold (0.3)       │
    │ If 0.85 >= 0.3: MATCHED ✓  │
    └─────────────┬───────────────┘
        │
        ▼
    Segregation Result
    {
      "Ticket_Number": "INC0010001",
      "Category": "Database Backup",
      "Confidence": 0.85,
      "Status": "Matched"
    }
        │
        ▼
    ┌──────────────────────────┐
    │ Team Assignment Lookup   │
    │ (from TEAM_MAPPING)      │
    └──────────┬───────────────┘
        │
        ▼
    Assignment Details
    {
      "Assigned_To": "dba_lead@company.com",
      "Assignment_Group": "DBA_Group",
      "Team": "DBA Team",
      "Priority": "critical"
    }
        │
        ▼
    ┌──────────────────────────┐
    │ Update ServiceNow        │
    │ (PATCH incident)         │
    └──────────┬───────────────┘
        │
        ▼
    OUTPUT (ServiceNow Updated)
    Ticket INC0010001 now assigned to dba_lead@company.com
    Change log entry created
    Assignment complete ✓

```

---

## 5. KEY PARAMETERS & THRESHOLDS

```
┌─────────────────────────────────────────┐
│ CONFIGURATION PARAMETERS                │
└─────────────────────────────────────────┘

ServiceNow Connection
├─ INSTANCE_URL: "https://dev12345.service-now.com"
├─ USERNAME: "your_username"
├─ PASSWORD: "your_password (or PAT)"
└─ TIMEOUT: 30 seconds

Segregation
├─ CONFIDENCE_THRESHOLD: 0.3  (Match if score >= 0.3)
├─ TF-IDF max_features: 100   (Max vocabulary size)
├─ Ngram range: (1, 2)        (Unigrams + bigrams)
└─ Similarity metric: Cosine  (between vectors)

API Limits
├─ TICKET_LIMIT: 100 (max per fetch)
├─ CATEGORY_LIMIT: 100
├─ BATCH_SIZE: 100 (pagination)
└─ RATE_LIMIT_BACKOFF: Auto

Team Mapping
├─ Categories: 15 (pre-configured)
├─ Teams: 15 (one per category)
├─ Priority Levels: critical, high, medium, low
└─ Support DRY-RUN mode: YES

Output
├─ Excel file: segregated_tickets_output.xlsx
├─ Sheets per category: Yes (31-char name limit)
├─ Unknown sheet: Yes (manual review)
└─ Summary sheet: Yes (statistics)

```

---

## 6. EXECUTION MODES

```
┌──────────────────────────────────────────────┐
│ THREE EXECUTION MODES                        │
└──────────────────────────────────────────────┘

MODE 1: MOCK DATA (No API, Testing)
┌──────────────────────────────────┐
│ $ python main.py --mock           │
│                                  │
│ • Generates 80 fake tickets      │
│ • Uses 15 sample categories      │
│ • Segregates & assigns locally   │
│ • No ServiceNow connection needed │
│ • Fast testing (< 1 second)      │
│ • Perfect for: POC, Testing      │
└──────────────────────────────────┘

MODE 2: DRY-RUN (Real API, Simulate)
┌──────────────────────────────────┐
│ $ python main.py --dry-run        │
│                                  │
│ • Connects to real ServiceNow     │
│ • Fetches real tickets & cats    │
│ • Segregates & assigns           │
│ • LOGS what would be done        │
│ • NO updates to ServiceNow       │
│ • Safe testing: Verify before live│
│ • Perfect for: Verification      │
└──────────────────────────────────┘

MODE 3: LIVE (Real API, Full Automation)
┌──────────────────────────────────┐
│ $ python main.py                  │
│                                  │
│ • Connects to real ServiceNow     │
│ • Fetches real tickets & cats    │
│ • Segregates & assigns           │
│ • UPDATES tickets in ServiceNow   │
│ • Creates audit trail            │
│ • Sends results to users         │
│ • Perfect for: Production         │
└──────────────────────────────────┘

```

---

## 7. RATE LIMITING & PAGINATION

```
┌─────────────────────────────────────────┐
│ HANDLING LARGE DATASETS                 │
└─────────────────────────────────────────┘

PAGINATION (Fetch 1000 tickets)
┌─────────────────────────────────────┐
│ Request #1: offset=0,   limit=100   │
│ Response: 100 tickets               │
│                                     │
│ Request #2: offset=100, limit=100   │
│ Response: 100 tickets               │
│                                     │
│ Request #3: offset=200, limit=100   │
│ Response: 100 tickets               │
│                                     │
│ ...                                 │
│                                     │
│ Request #10: offset=900, limit=100  │
│ Response: 100 tickets               │
│                                     │
│ Total: 1000 tickets fetched         │
└─────────────────────────────────────┘

RATE LIMITING (ServiceNow API)
┌─────────────────────────────────────┐
│ ServiceNow Default: 100 req/min     │
│                                     │
│ Requests sent:      ████████░░░░░░  │
│ Time remaining:     30 seconds      │
│                                     │
│ Rate limit status:  95 requests left│
│ Next retry time:    35 seconds      │
│                                     │
│ When limit approaches:              │
│ └─ Auto-backoff: Sleep for N sec   │
│ └─ Retry after reset               │
└─────────────────────────────────────┘

```

---

## 8. REAL-WORLD EXAMPLE

```
Scenario: Company processes 1000 tickets daily

TIMELINE:
├─ 08:00 AM
│  └─ Admin runs: python main.py
│
├─ 08:00-08:02 AM (120 seconds)
│  ├─ Fetch 1000 tickets (paginated, 10 requests)
│  ├─ Fetch 15 categories
│  └─ Initialize engine
│
├─ 08:02-08:05 AM (180 seconds)
│  ├─ Segregate 1000 tickets via TF-IDF
│  │  └─ Match: 750 tickets
│  │  └─ Unknown: 250 tickets
│  └─ Assign 750 to teams
│
├─ 08:05-08:07 AM (120 seconds)
│  ├─ Update 750 tickets in ServiceNow
│  │  └─ PATCH /api/now/table/incident (750 requests)
│  └─ Create change log entries
│
├─ 08:07-08:08 AM (60 seconds)
│  ├─ Generate report
│  ├─ Export to Excel
│  └─ Send notifications
│
└─ 08:08 AM
   └─ COMPLETE ✓
     • 750 tickets automatically assigned
     • 250 unknown tickets flagged for review
     • Full audit trail in ServiceNow
     • Excel export ready for download

COST SAVINGS:
• Manual assignment: 8 hours / 1000 = 0.03 hours per ticket
• Automated: < 1 minute for all 1000 tickets
• Time saved: 7+ hours per day
• Accuracy: TF-IDF matching eliminates human error
```

---

## Summary

The **ServiceNow API Automation** flow is:

1. **Authenticate** → Connect to ServiceNow API
2. **Fetch** → Pull tickets & categories with pagination
3. **Convert** → Transform API responses to DataFrames
4. **Segregate** → Match tickets to categories via TF-IDF
5. **Assign** → Lookup team mappings from config
6. **Update** → PATCH tickets in ServiceNow (or dry-run)
7. **Audit** → Create change log entries
8. **Report** → Export results to Excel & display stats

All with optional **dry-run mode** for safe testing! 🚀
