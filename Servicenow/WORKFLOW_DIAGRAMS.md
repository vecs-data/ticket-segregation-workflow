# SmartOps Complete Workflow Diagrams

## 1. MAIN WORKFLOW - OVERVIEW

```
┌─────────────────────────────────────────────────────────────────────┐
│                     SMARTOPS COMPLETE WORKFLOW                      │
└─────────────────────────────────────────────────────────────────────┘

                              START
                                │
                    ┌───────────▼───────────┐
                    │  Load Configuration   │
                    │  • API credentials    │
                    │  • Team mapping       │
                    │  • Thresholds (0.3)   │
                    └───────────┬───────────┘
                                │
                    ┌───────────▼───────────┐
                    │ Initialize Clients    │
                    │  • ServiceNow         │
                    │  • OpenAI (optional)  │
                    └───────────┬───────────┘
                                │
                    ┌───────────▼───────────┐
                    │ Test Connection       │
                    │ to ServiceNow API     │
                    └───────────┬───────────┘
                                │
                        ┌───────▼────────┐
                        │ Connected?     │
                        └───────┬────────┘
                             NO │ YES
                        ┌───────┘│
                        │        ▼
                        │   ┌────────────────────┐
                        │   │ FETCH PHASE        │
                        │   │ • Tickets (100)    │
                        │   │ • Categories (15)  │
                        │   │ • Pagination       │
                        │   └────────┬───────────┘
                        │            │
                        │   ┌────────▼───────────┐
                        │   │ SEGREGATION PHASE  │
                        │   │ • TF-IDF Match     │
                        │   │ • 100 iterations   │
                        │   │ • Confidence score │
                        │   └────────┬───────────┘
                        │            │
                        │   ┌────────▼───────────┐
                        │   │ DECISION TREE      │
                        │   │ Score >= 0.3?      │
                        │   └────┬──────────┬────┘
                        │       YES       NO
                        │        │         │
                        │   ┌────▼─┐  ┌──▼──────────┐
                        │   │Match │  │Optional LLM │
                        │   │ (75) │  │Validate (25)│
                        │   └────┬─┘  └──┬──────────┘
                        │        │       │
                        │   ┌────▼───────▼────────┐
                        │   │ TEAM ASSIGNMENT     │
                        │   │ Lookup TEAM_MAPPING │
                        │   └────────┬────────────┘
                        │            │
                        │   ┌────────▼────────────┐
                        │   │ SERVICENOW UPDATE   │
                        │   │ Dry-Run or Live?    │
                        │   └────┬──────────┬─────┘
                        │       DRY          LIVE
                        │        │            │
                        │   ┌────▼─┐    ┌───▼──────────┐
                        │   │  Log │    │PATCH incident│
                        │   │Output│    │Create audit  │
                        │   └────┬─┘    └───┬──────────┘
                        │        │          │
                        │   ┌────▼──────────▼────┐
                        │   │ REPORTING & EXPORT │
                        │   │ Excel + Summary    │
                        │   └────────┬───────────┘
                        │            │
                        │            ▼
                        │          END ✓
                        │
                        └──→ ERROR HANDLING
                            └─ Log & Retry
```

---

## 2. DETAILED PHASE-BY-PHASE FLOW

### PHASE 1: INITIALIZATION & AUTHENTICATION

```
┌─────────────────────────────────────────────────────────────┐
│ PHASE 1: INITIALIZATION & AUTHENTICATION                    │
└─────────────────────────────────────────────────────────────┘

USER STARTS:
python main.py [--mock] [--dry-run]
         │
         ▼
┌──────────────────────────────────┐
│ Read Configuration Files         │
├──────────────────────────────────┤
│ FROM:                            │
│ • config.py (credentials)        │
│ • .env (API keys)                │
│ • TEAM_MAPPING (team assignments)│
│                                  │
│ EXTRACTED:                       │
│ • instance_url                   │
│ • username                       │
│ • password                       │
│ • openai_api_key (if LLM=True)   │
│ • 15 team mappings               │
└────────────────┬─────────────────┘
                 │
         ┌───────▼────────┐
         │ Create Clients │
         ├────────────────┤
         │ • ServiceNow   │
         │   Client      │
         │ • OpenAI      │
         │   Client      │
         │   (optional)   │
         └───────┬────────┘
                 │
         ┌───────▼────────────────┐
         │ Test Connection        │
         ├────────────────────────┤
         │ GET /api/now/sys_user  │
         │ Expected: 200 OK       │
         │                        │
         │ Response:              │
         │ {                      │
         │   "result": [{...}]    │
         │ }                      │
         └───────┬────────────────┘
                 │
            ┌────▼────┐
            │Success?  │
            └────┬─────┘
                 │
             ┌───┴────┐
            YES      NO
             │        │
             ▼        ▼
         CONTINUE  ERROR
                    STOP
```

---

### PHASE 2: FETCH DATA

```
┌─────────────────────────────────────────────────────────────┐
│ PHASE 2: FETCH DATA FROM SERVICENOW (With Pagination)       │
└─────────────────────────────────────────────────────────────┘

REQUEST #1 (Tickets)
┌──────────────────────────────────┐
│ GET /api/now/table/incident      │
│ Parameters:                      │
│  • sysparm_query=state=1         │
│  • sysparm_limit=100             │
│  • sysparm_offset=0              │
│  • sysparm_fields=number,short_  │
│    description,description       │
└────────────┬─────────────────────┘
             │
             ▼
┌──────────────────────────────────┐
│ Response: 100 Tickets            │
├──────────────────────────────────┤
│ {                                │
│   "result": [                    │
│     {                            │
│       "number": "INC0010001",    │
│       "short_description": "...",│
│       "description": "...",      │
│       "state": "1"               │
│     },                           │
│     ... (99 more)                │
│   ]                              │
│ }                                │
└────────────┬─────────────────────┘
             │
     ┌───────▼────────┐
     │ Has more?      │
     │ (Check count)  │
     └───────┬────────┘
            YES │ NO
        ┌───────┘│
        │        ▼
        │   ┌─────────────────────┐
        │   │ Store in tickets_df │
        │   │ (100 tickets total) │
        │   └──────────┬──────────┘
        │              │
        │   ┌──────────▼────────────┐
        │   │ REQUEST #2: Categories│
        │   │ GET /api/now/table/   │
        │   │ incident_category     │
        │   │ sysparm_limit=100     │
        │   └──────────┬────────────┘
        │              │
        │              ▼
        │   ┌──────────────────────┐
        │   │ Response: 15 Category │
        │   │                      │
        │   │ {                    │
        │   │   "result": [        │
        │   │     {               │
        │   │       "sys_id":"...",│
        │   │       "name":"DB...",│
        │   │       "desc":"..."   │
        │   │     },              │
        │   │     ... (14 more)    │
        │   │   ]                 │
        │   │ }                    │
        │   └──────────┬───────────┘
        │              │
        │   ┌──────────▼──────────┐
        │   │ Store in categories │
        │   │ (15 categories)     │
        │   └────────┬────────────┘
        │            │
        └────────────┘
                 │
             ┌───▼─────────────────┐
             │ READY FOR MATCHING  │
             └─────────────────────┘
```

---

### PHASE 3: SEGREGATION ENGINE (TF-IDF MATCHING)

```
┌──────────────────────────────────────────────────────────────┐
│ PHASE 3: TICKET SEGREGATION (TF-IDF)                         │
│ Loop: For each of 100 tickets                                │
└──────────────────────────────────────────────────────────────┘

ITERATION 1 (Ticket: INC0010001)
┌────────────────────────────────────────┐
│ Ticket Data                            │
├────────────────────────────────────────┤
│ Number: INC0010001                     │
│ Short Desc: "Database down"            │
│ Description: "Production DB unreachable│
│              for 30 minutes"           │
└────────────┬─────────────────────────┘
             │
             ▼
┌────────────────────────────────────────┐
│ STEP 1: PREPROCESS TEXT                │
├────────────────────────────────────────┤
│ Original:                              │
│ "Database down - Production DB ..."    │
│                                        │
│ Cleaned:                               │
│ "database down production db unreachable│
│  for 30 minutes"                       │
│                                        │
│ Operations:                            │
│ • Lowercase                            │
│ • Remove special chars                 │
│ • Tokenize (split into words)          │
└────────────┬─────────────────────────┘
             │
             ▼
┌────────────────────────────────────────┐
│ STEP 2: TF-IDF VECTORIZATION           │
├────────────────────────────────────────┤
│ Vectorizer Setup:                      │
│ TfidfVectorizer(                       │
│   max_features=100,                    │
│   ngram_range=(1, 2)  # unigrams+     │
│ )                                      │
│                                        │
│ Input Texts:                           │
│ • 15 category descriptions             │
│ • 1 ticket description                 │
│ Total: 16 texts                        │
│                                        │
│ Output Vectors:                        │
│ • Ticket: [0.2, 0.5, 0.1, ...] (100D) │
│ • Cat 1:  [0.1, 0.3, 0.4, ...] (100D) │
│ • Cat 2:  [0.4, 0.2, 0.3, ...] (100D) │
│ • ...                                  │
│ • Cat 15: [0.3, 0.5, 0.2, ...] (100D) │
└────────────┬─────────────────────────┘
             │
             ▼
┌────────────────────────────────────────┐
│ STEP 3: COSINE SIMILARITY              │
├────────────────────────────────────────┤
│ For each category, compute:            │
│ similarity = cos(ticket_vec, cat_vec)  │
│                                        │
│ Results:                               │
│ Category 1 (DB Admin):     0.85  ← MAX│
│ Category 2 (Network):      0.12      │
│ Category 3 (Hardware):     0.05      │
│ Category 4 (VPN):          0.08      │
│ ...                                    │
│ Category 15 (Other):       0.02      │
└────────────┬─────────────────────────┘
             │
             ▼
┌────────────────────────────────────────┐
│ STEP 4: DECISION                       │
├────────────────────────────────────────┤
│ Best match: Category 1                 │
│ Confidence: 0.85 (85%)                 │
│                                        │
│ Compare vs Threshold:                  │
│ 0.85 >= 0.3?  YES ✓                    │
│                                        │
│ Action: MATCH                          │
└────────────┬─────────────────────────┘
             │
             ▼
┌────────────────────────────────────────┐
│ STEP 5: STORE RESULT                   │
├────────────────────────────────────────┤
│ {                                      │
│   "Ticket_Number": "INC0010001",       │
│   "Short_Description": "Database...",  │
│   "Matched_Category": "DB Admin",      │
│   "Confidence_Score": 0.85,            │
│   "Status": "MATCHED ✓"                │
│ }                                      │
│                                        │
│ Store in: segregated_data["DB Admin"]  │
└────────────┬─────────────────────────┘
             │
      ┌──────▼──────────┐
      │ Iteration 2     │
      │ (INC0010002)    │
      │ ...             │
      │ Iteration 100   │
      │ (INC0010100)    │
      └──────┬──────────┘
             │
             ▼
    ┌─────────────────────┐
    │ SEGREGATION COMPLETE│
    │ 75 Matched    ✓     │
    │ 25 Unknown    ⚠     │
    └─────────────────────┘
```

---

### PHASE 4: OPTIONAL LLM VALIDATION

```
┌───────────────────────────────────────────────────────────┐
│ PHASE 4: OPTIONAL LLM VALIDATION (GPT-3.5-Turbo)          │
│ Only processes 25 UNKNOWN tickets                         │
└───────────────────────────────────────────────────────────┘

INPUT: Unknown Tickets (Confidence < 0.3)
┌──────────────────────────────────────┐
│ Sample from Unknown (limit to 5)     │
├──────────────────────────────────────┤
│ 1. INC0010003: "License issue"       │
│ 2. INC0010004: "Slow performance"    │
│ 3. INC0010011: "Cannot connect"      │
│ 4. INC0010025: "Data corruption"     │
│ 5. INC0010047: "Permission denied"   │
└────────────────┬─────────────────────┘
                 │
        ┌────────▼─────────┐
        │ For each sample: │
        └────────┬─────────┘
                 │
        ┌────────▼──────────────────┐
        │ CONSTRUCT PROMPT           │
        ├────────────────────────────┤
        │ System: You are a support  │
        │ ticket classifier.         │
        │                            │
        │ User:                      │
        │ "Given 15 categories:      │
        │ [DB Admin, Network, ...]   │
        │                            │
        │ Categorize this ticket:    │
        │ 'License issue - User      │
        │  cannot activate software' │
        │                            │
        │ Respond with JSON:         │
        │ {"category": "...",        │
        │  "reason": "..."}"         │
        └────────┬──────────────────┘
                 │
        ┌────────▼──────────────────┐
        │ CALL OPENAI API            │
        ├────────────────────────────┤
        │ POST https://api.openai... │
        │                            │
        │ Request:                   │
        │ {                          │
        │   "model": "gpt-3.5-turbo",│
        │   "messages": [...],       │
        │   "max_tokens": 100,       │
        │   "temperature": 0.3       │
        │ }                          │
        │                            │
        │ Latency: 2-5 seconds       │
        │ Cost: ~$0.002              │
        └────────┬──────────────────┘
                 │
        ┌────────▼──────────────────┐
        │ PARSE RESPONSE             │
        ├────────────────────────────┤
        │ {                          │
        │   "category": "Licensing", │
        │   "reason": "Software     │
        │    activation issue"       │
        │ }                          │
        │                            │
        │ ✓ Valid JSON               │
        │ ✓ Category in list         │
        │ ✓ Reason provided          │
        └────────┬──────────────────┘
                 │
        ┌────────▼──────────────────┐
        │ ATTACH TO TICKET           │
        ├────────────────────────────┤
        │ ticket[                    │
        │   'LLM_Suggested_Category' │
        │ ] = "Licensing"            │
        │                            │
        │ ticket[                    │
        │   'LLM_Reason'             │
        │ ] = "Software activation"  │
        └────────┬──────────────────┘
                 │
        ┌────────▼──────────────────┐
        │ PRINT SUGGESTION           │
        ├────────────────────────────┤
        │ "INC0010003:               │
        │  LLM Suggests: Licensing   │
        │  Reason: Software activ..." │
        └────────┬──────────────────┘
                 │
         ┌───────▼────────┐
         │ Next sample... │
         │ (up to 5)      │
         └───────┬────────┘
                 │
                 ▼
        ┌─────────────────────────┐
        │ VALIDATION COMPLETE     │
        │ 5 suggestions provided  │
        │ Ready for manual review │
        └─────────────────────────┘
```

---

### PHASE 5: TEAM ASSIGNMENT

```
┌────────────────────────────────────────────────────────┐
│ PHASE 5: TEAM ASSIGNMENT                               │
│ Map categories → teams/users                           │
└────────────────────────────────────────────────────────┘

TEAM_MAPPING Configuration
┌─────────────────────────────────────────┐
│ {                                       │
│   "DB Admin": {                         │
│     "team": "DBA Team",                 │
│     "assigned_to": "dba_lead@...",      │
│     "assignment_group": "DBA_Group",    │
│     "priority": "critical"              │
│   },                                    │
│   "Network": {                          │
│     "team": "Network Team",             │
│     "assigned_to": "network@...",       │
│     "assignment_group": "NET_Group",    │
│     "priority": "high"                  │
│   },                                    │
│   ... (13 more categories)              │
│ }                                       │
└────────────┬──────────────────────────┘
             │
    ┌────────▼──────────────────┐
    │ FOR EACH MATCHED TICKET    │
    │ (75 total)                 │
    └────────┬──────────────────┘
             │
    ┌────────▼──────────────────┐
    │ LOOKUP ASSIGNMENT          │
    ├────────────────────────────┤
    │ Ticket Category: "DB Admin"│
    │ Found in mapping: YES ✓    │
    │                            │
    │ Extract:                   │
    │ • assigned_to:             │
    │   "dba_lead@company.com"   │
    │ • assignment_group:        │
    │   "DBA_Group"              │
    │ • priority: "critical"     │
    └────────┬──────────────────┘
             │
    ┌────────▼──────────────────┐
    │ CREATE ASSIGNMENT RECORD   │
    ├────────────────────────────┤
    │ {                          │
    │   "Ticket_Number":         │
    │     "INC0010001",          │
    │   "Assigned_To":           │
    │     "dba_lead@...",        │
    │   "Assignment_Group":      │
    │     "DBA_Group",           │
    │   "Team": "DBA Team",      │
    │   "Priority": "critical",  │
    │   "Assignment_Status":     │
    │     "Ready to Update"      │
    │ }                          │
    └────────┬──────────────────┘
             │
    ┌────────▼──────────────────┐
    │ NEXT TICKET (76 total)    │
    │ ...                        │
    └────────┬──────────────────┘
             │
             ▼
    ┌─────────────────────────┐
    │ ALL 75 ASSIGNED ✓       │
    │ READY FOR SERVICENOW    │
    │ UPDATE                  │
    └─────────────────────────┘
```

---

### PHASE 6: SERVICENOW UPDATE

```
┌──────────────────────────────────────────────────────────┐
│ PHASE 6: SERVICENOW UPDATE (Live or Dry-Run)             │
└──────────────────────────────────────────────────────────┘

CHECK: Dry-Run Mode?
┌────────────────────┐
│ if dry_run == True │
└────────┬───────────┘
        YES │ NO
    ┌───────┴────────┐
    │                │
    ▼                ▼

DRY-RUN              LIVE UPDATE
(SIMULATION)         (REAL)

┌──────────────────┐ ┌──────────────────────┐
│ FOR EACH TICKET: │ │ FOR EACH TICKET:     │
├──────────────────┤ ├──────────────────────┤
│ PRINT:           │ │ PATCH /api/now/table/│
│ "[DRY RUN]       │ │ incident             │
│  Would update    │ │                      │
│  INC0010001:     │ │ Query Param:         │
│  assigned_to=... │ │ sysparm_query=      │
│  group=..."      │ │ number=INC0010001    │
│                  │ │                      │
│ Action: LOGGED   │ │ Body:                │
│ ✓ Log file       │ │ {                    │
│ ✓ Console        │ │   "assigned_to":     │
│ ✓ No changes     │ │   "dba_lead@...",   │
│                  │ │   "assignment_group":│
│                  │ │   "DBA_Group"       │
│                  │ │ }                    │
│                  │ │                      │
│                  │ │ Response: 200 OK ✓  │
│                  │ │ {                    │
│                  │ │   "result": {        │
│                  │ │     "number":        │
│                  │ │     "INC0010001",    │
│                  │ │     "assigned_to":   │
│                  │ │     "dba_lead@...",  │
│                  │ │     "updated":       │
│                  │ │     "2025-12-02T..." │
│                  │ │   }                  │
│                  │ │ }                    │
│                  │ │                      │
│                  │ │ ✓ Update successful │
├──────────────────┤ ├──────────────────────┤
│ NEXT TICKET:     │ │ CREATE AUDIT LOG:    │
│ (75 total)       │ │                      │
└──────────────────┘ │ POST /api/now/table/ │
                     │ cmdb_change_log      │
                     │                      │
                     │ Body:                │
                     │ {                    │
                     │   "ticket_number":   │
                     │   "INC0010001",      │
                     │   "category":        │
                     │   "DB Admin",        │
                     │   "confidence":      │
                     │   "85%",             │
                     │   "change_type":     │
                     │   "Automated        │
                     │    Segregation"      │
                     │ }                    │
                     │                      │
                     │ Response: 201        │
                     │ Created ✓            │
                     │                      │
                     │ ✓ Audit entry created
                     ├──────────────────────┤
                     │ NEXT TICKET:         │
                     │ (75 total)           │
                     └──────────────────────┘
         │                   │
         └─────────┬─────────┘
                   │
             ┌─────▼──────┐
             │ COMPLETE   │
             │ 75 Updated │
             │ All Audited│
             └────────────┘
```

---

### PHASE 7: REPORTING & EXPORT

```
┌──────────────────────────────────────────────────┐
│ PHASE 7: REPORTING & EXPORT TO EXCEL             │
└──────────────────────────────────────────────────┘

AGGREGATE RESULTS
┌────────────────────────────────────┐
│ Total Tickets Processed: 100       │
│ Matched: 75 (75%)                  │
│ Unknown: 25 (25%)                  │
│ Updated in ServiceNow: 75          │
│ Failed Updates: 0                  │
│ LLM Suggestions: 5                 │
└────────────┬───────────────────────┘
             │
             ▼
┌──────────────────────────────────────┐
│ CREATE EXCEL WORKBOOK                │
│ segregated_tickets_output.xlsx       │
├──────────────────────────────────────┤
│                                      │
│ Sheet 1: SUMMARY                     │
│ ┌──────────────────────────────────┐│
│ │ Metric           | Value         ││
│ ├──────────────────────────────────┤│
│ │ Total Processed  | 100           ││
│ │ Matched          | 75 (75%)      ││
│ │ Unknown          | 25 (25%)      ││
│ │ Updated          | 75            ││
│ │ Failed           | 0             ││
│ │ Duration         | 2m 34s        ││
│ └──────────────────────────────────┘│
│                                      │
│ Sheet 2: DB_Admin (15 tickets)       │
│ ┌──────────────────────────────────┐│
│ │ INC#  | Description | Confidence ││
│ │ 10001 | Database... | 0.85       ││
│ │ 10005 | Backup...   | 0.78       ││
│ │ ...   | ...         | ...        ││
│ └──────────────────────────────────┘│
│                                      │
│ Sheet 3: Network (12 tickets)        │
│ ┌──────────────────────────────────┐│
│ │ INC#  | Description | Confidence ││
│ │ 10002 | VPN issue   | 0.82       ││
│ │ 10008 | Latency...  | 0.75       ││
│ │ ...   | ...         | ...        ││
│ └──────────────────────────────────┘│
│                                      │
│ ... (13 more category sheets)        │
│                                      │
│ Sheet 30: Unknown (25 tickets)       │
│ ┌──────────────────────────────────┐│
│ │ INC#  | Description | LLM Suggest││
│ │ 10003 | License...  | Licensing  ││
│ │ 10004 | Performance | Hardware   ││
│ │ ...   | ...         | ...        ││
│ └──────────────────────────────────┘│
│                                      │
└──────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────┐
│ PRINT CONSOLE SUMMARY                │
├──────────────────────────────────────┤
│ ✓ Segregation Complete!              │
│                                      │
│ Results:                             │
│   • Loaded: 100 tickets              │
│   • Matched: 75 tickets (75%)        │
│   • Unknown: 25 tickets (25%)        │
│   • Updated: 75 tickets              │
│   • Failed: 0                        │
│   • Processing Time: 2m 34s          │
│                                      │
│ Top Categories by Count:             │
│   1. Unknown: 25 tickets             │
│   2. DB Admin: 15 tickets            │
│   3. Network: 12 tickets             │
│   4. Hardware: 10 tickets            │
│   5. User Access: 8 tickets          │
│   ... (10 more)                      │
│                                      │
│ Output File:                         │
│   segregated_tickets_output.xlsx ✓   │
│                                      │
│ Audit Trail:                         │
│   • ServiceNow change logs: 75       │
│   • Timestamp: 2025-12-02 10:34:45   │
│                                      │
└──────────────────────────────────────┘
             │
             ▼
          END ✓
```

---

## 3. DECISION TREE (Comprehensive)

```
┌──────────────────────────────────────────────┐
│          TICKET PROCESSING DECISION TREE      │
└──────────────────────────────────────────────┘

                    TICKET
                      │
                      ▼
         ┌────────────────────────┐
         │ TF-IDF Match Attempted │
         └────────────┬───────────┘
                      │
           ┌──────────▼──────────┐
           │ Confidence Score:   │
           │ (0.0 to 1.0)        │
           └──────────┬──────────┘
                      │
         ┌────────────┼────────────┐
         │            │            │
      0.8+         0.3-0.8       <0.3
         │            │            │
         ▼            ▼            ▼
    ┌────────┐  ┌─────────┐  ┌──────────┐
    │MATCHED │  │ MEDIUM  │  │ UNKNOWN  │
    │ HIGH   │  │CONFIDENCE   │ (NO TF)  │
    │CONF ✓  │  │         │  │          │
    └────────┘  └─────────┘  └──────────┘
         │            │            │
         │            │            ▼
         │            │       ┌─────────────┐
         │            │       │ Enable LLM? │
         │            │       │ (Optional)  │
         │            │       └──┬────┬─────┘
         │            │         YES  NO
         │            │          │    │
         │            │          ▼    ▼
         │            │   ┌──────┐ ┌────────┐
         │            │   │CALL  │ │SKIP    │
         │            │   │GPT   │ │LLM     │
         │            │   └──┬───┘ └───┬────┘
         │            │      │         │
         │            │      ▼         ▼
         │            │  ┌────────┐ ┌───────┐
         │            │  │LLM     │ │MARK   │
         │            │  │SUGGEST │ │UNKNOWN│
         │            │  └───┬────┘ └───┬───┘
         │            │      │          │
         ▼            ▼      ▼          ▼
    ┌──────────────────────────────────────┐
    │    CATEGORIZED TICKET                │
    │                                      │
    │  Found Category: <CATEGORY_NAME>     │
    │  Confidence: <SCORE>                 │
    │  Status: MATCHED/SUGGESTED/UNKNOWN   │
    └────────────┬─────────────────────────┘
                 │
         ┌───────▼──────────┐
         │ Category in      │
         │ TEAM_MAPPING?    │
         └───────┬──────────┘
              YES│NO
           ┌─────┴─────┐
           │           │
           ▼           ▼
      ┌────────┐  ┌──────────┐
      │ASSIGN  │  │NO TEAM   │
      │TO TEAM │  │MAPPING   │
      │(auto)  │  │(manual)  │
      └───┬────┘  └────┬─────┘
          │            │
          ▼            ▼
       ┌──────────────────────┐
       │ READY FOR UPDATE     │
       │ (Live or Dry-Run)    │
       └──────────┬───────────┘
                  │
          ┌───────▼────────┐
          │ Update Mode?   │
          └───────┬────────┘
              LIVE│DRY
           ┌──────┴──────┐
           │             │
           ▼             ▼
       ┌────────┐   ┌──────────┐
       │PATCH   │   │LOG ONLY  │
       │SERVICENOW  │(Simulation)
       │+ AUDIT │   │          │
       │LOG ✓   │   └──────────┘
       └────────┘
```

---

## 4. ERROR HANDLING & FALLBACK FLOW

```
┌──────────────────────────────────────────────┐
│       ERROR HANDLING & FALLBACK FLOW          │
└──────────────────────────────────────────────┘

                    ERROR EVENT
                        │
                        ▼
            ┌──────────────────────┐
            │ Classify Error Type  │
            └──────────┬───────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
    API_ERROR    LLM_ERROR    PARSING_ERROR
        │              │              │
        ▼              ▼              ▼
    ┌─────────┐  ┌──────────┐  ┌──────────┐
    │Connection   │No API Key │ │Invalid   │
    │Timeout      │Rate Limit │ │JSON      │
    │Rate Limit   │No Response│ │Truncated │
    └─────┬───┘  └────┬─────┘  └────┬─────┘
          │           │             │
          ▼           ▼             ▼
    ┌──────────────────────────────────────┐
    │ RETRY LOGIC                          │
    ├──────────────────────────────────────┤
    │ Max Retries: 3                       │
    │ Backoff: Exponential (1s, 2s, 4s)    │
    └──────────────┬───────────────────────┘
                   │
        ┌──────────▼──────────┐
        │ Attempt Count <= 3? │
        └──────────┬──────────┘
             YES │ NO
        ┌────────┘│
        │         ▼
        │     ┌─────────────┐
        │     │SKIP & LOG   │
        │     │ERROR (Final)│
        │     └─────┬───────┘
        ▼           │
    ┌──────────┐    │
    │SLEEP     │    │
    │(backoff) │    │
    └────┬─────┘    │
         │          │
         ▼          │
    ┌──────────┐    │
    │RETRY     │    │
    │REQUEST   │    │
    └────┬─────┘    │
         │          │
    ┌────▼──────────┴──────┐
    │                      │
    │    Continue or Fail  │
    └──────────────────────┘
```

---

## 5. EXECUTION MODES FLOWCHART

```
┌──────────────────────────────────────────────┐
│         EXECUTION MODES (Three Paths)         │
└──────────────────────────────────────────────┘

            COMMAND LINE INPUT
                   │
        ┌──────────┼──────────┐
        │          │          │
        ▼          ▼          ▼
    python     python      python
    main.py    main.py     main.py
    --mock     --dry-run   (live)
        │          │          │
        ▼          ▼          ▼
    ┌──────┐  ┌───────┐  ┌───────┐
    │ MOCK │  │DRY-RUN│  │ LIVE  │
    └──┬───┘  └───┬───┘  └───┬───┘
       │          │          │
       ▼          ▼          ▼
   ┌────────────────────────────────────────┐
   │ DATA SOURCE                            │
   ├────────────────────────────────────────┤
   │ MOCK:     Generate fake tickets (local)│
   │ DRY-RUN:  Fetch real, don't update     │
   │ LIVE:     Fetch real, update SNOW      │
   └──┬───────────┬────────────┬────────────┘
      │           │            │
      ▼           ▼            ▼
   ┌─────────────────────────────────────┐
   │ SEGREGATION ENGINE (Same)           │
   │ • TF-IDF Matching                   │
   │ • Optional LLM Validation           │
   │ • Team Assignment Lookup            │
   └──┬───────────┬────────────┬─────────┘
      │           │            │
      ▼           ▼            ▼
   ┌─────────────────────────────────────┐
   │ UPDATE PHASE (Different!)           │
   ├─────────────────────────────────────┤
   │ MOCK:     None (data local)         │
   │ DRY-RUN:  Log what would update     │
   │ LIVE:     PATCH /api/now/incident   │
   │           + POST change log         │
   └──┬───────────┬────────────┬─────────┘
      │           │            │
      ▼           ▼            ▼
   ┌─────────────────────────────────────┐
   │ REPORTING (Same - Excel Export)     │
   │ • Summary sheet                     │
   │ • Per-category sheets               │
   │ • Unknown sheet                     │
   │ • Console output                    │
   └─────────────────────────────────────┘
```

---

## Summary: Key Flows at a Glance

| **Phase** | **Input** | **Process** | **Output** |
|-----------|-----------|-------------|-----------|
| 1. Init | Config | Load creds, test API | Client ready |
| 2. Fetch | ServiceNow API | Paginate, rate-limit | 100 tickets + 15 categories |
| 3. Segment | Ticket descriptions | TF-IDF vectorize, cosine sim | Matched (75) + Unknown (25) |
| 4. LLM (opt) | Unknown tickets | GPT-3.5 prompt + parse | Suggestions + reasoning |
| 5. Assign | Categories | TEAM_MAPPING lookup | Team/user/group per ticket |
| 6. Update | Assignments | PATCH incident, POST audit | ServiceNow updated + logs |
| 7. Report | All results | Aggregate, export Excel | Excel file + console summary |

All diagrams show **real data flow, actual API calls, and decision points** from the SmartOps workflow. 🚀
