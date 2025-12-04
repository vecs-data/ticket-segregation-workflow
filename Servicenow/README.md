# ServiceNow API Automation Project

## Overview

This sub-project extends the ticket segregation workflow to work directly with ServiceNow API, eliminating the need for manual Excel file uploads. The automation workflow:

1. **Fetches** tickets and categories directly from ServiceNow API
2. **Segregates** tickets using TF-IDF matching (reuses engine from `ticket_segregation_workflow`)
3. **Assigns** segregated tickets to teams automatically
4. **Updates** ServiceNow with assignments and audit logs
5. **Provides** a Streamlit UI for review, approval, and monitoring

## Project Structure

```
servicenow_api_automation/
├── servicenow_client.py          # ServiceNow API client (auth, pagination, rate limiting)
├── ticket_segregation_engine.py  # Core TF-IDF matching engine (works with DataFrames)
├── task_assignment.py            # Team assignment logic and load balancing
├── workflow.py                   # Orchestrator (fetch → segregate → assign → update)
├── mock_servicenow_data.py       # Generate mock tickets/categories for testing
├── config.example.py             # Configuration template
├── streamlit_app.py              # Streamlit UI for monitoring and approval
├── tests/
│   ├── test_servicenow_client.py     # API client tests (mocked calls)
│   ├── test_segregation.py           # Segregation engine tests
│   ├── test_assignment.py            # Assignment logic tests
│   └── test_workflow.py              # End-to-end workflow tests
├── requirements.txt              # Dependencies
└── README.md                     # This file
```

## Quick Start

### 1. Install Dependencies

```bash
cd servicenow_api_automation
pip install -r requirements.txt
```

### 2. Configure ServiceNow Connection

Create a `.env` file in this directory:

```
SERVICENOW_INSTANCE_URL=https://dev12345.service-now.com
SERVICENOW_USERNAME=your_username
SERVICENOW_PASSWORD=your_password_or_pat
CONFIDENCE_THRESHOLD=0.3
OPENAI_API_KEY=sk-...  # Optional for LLM validation
```

Or copy and edit `config.example.py`:

```bash
cp config.example.py config.py
```

### 3. Test with Mock Data (No ServiceNow Required)

Generate mock data to test the workflow locally:

```bash
python mock_servicenow_data.py
# Generates: mock_tickets.json, mock_categories.json
```

### 4. Run the Workflow

```bash
python -m pytest -q          # Run tests
python main.py --dry-run     # Simulate (doesn't update ServiceNow)
python main.py               # Run with actual ServiceNow updates
```

### 5. Launch Streamlit UI

```bash
streamlit run streamlit_app.py
```

## Key Components

### ServiceNow Client (`servicenow_client.py`)

- Handles OAuth/Basic auth
- Manages pagination for large result sets
- Implements rate limiting (respects X-RateLimit-* headers)
- Provides methods:
  - `fetch_tickets(limit, query, fields)` — Fetch open incidents
  - `fetch_categories(limit, fields)` — Fetch ticket categories
  - `update_ticket(ticket_number, update_data)` — Update a ticket
  - `assign_ticket(ticket_number, assigned_to, assignment_group)` — Assign ticket
  - `create_change_log_entry(...)` — Audit trail in ServiceNow

### Ticket Segregation Engine (`ticket_segregation_engine.py`)

- Works with DataFrames (from API) or Excel files
- Uses TF-IDF + cosine similarity for matching
- Supports optional LLM validation for unknowns (OpenAI)
- Returns results with:
  - `Ticket_Number`, `Category`, `Confidence`, `Status`
  - Confidence threshold filters matches

### Task Assignment (`task_assignment.py`)

- Maps categories to teams via configuration
- Supports per-category assignment rules (assignee, group, priority)
- Tracks assignment counts for load balancing
- Validates team mappings

### Workflow Orchestrator (`workflow.py`)

- Orchestrates the complete pipeline: fetch → segregate → assign → update
- Dry-run mode for safe testing
- Collects and reports results
- Logs progress and errors

## Team Mapping Configuration

Define team assignments in `config.py`:

```python
TEAM_MAPPING = {
    "Pharmacy Operations": {
        "team": "Pharmacy Team",
        "assigned_to": "pharmacy_lead@company.com",
        "assignment_group": "Pharmacy_Group",
        "priority": "high"
    },
    "Inventory Sync": {
        "team": "Inventory Team",
        "assigned_to": "inventory_lead@company.com",
        "assignment_group": "Inventory_Group",
        "priority": "medium"
    },
    # ... more categories
}
```

## Running the Full Automation

### Dry-Run (Simulate, No Updates)

```bash
python main.py --dry-run
```

Output:
```
=============== SERVICENOW TICKET SEGREGATION WORKFLOW ===============

Fetched 120 tickets and 15 categories
Segregated 120 tickets (90 matched, 30 unknown)
[DRY RUN] Would assign 90 tickets to teams

=============== WORKFLOW RESULTS ===============
fetched_tickets: 120
segregated_tickets: 120
assigned_tickets: 90
failed_assignments: 0
timestamp: 2025-12-01T...
```

### Live Run (Updates ServiceNow)

```bash
python main.py
```

This will:
1. Fetch open tickets from ServiceNow
2. Segregate them by category
3. Assign tickets to teams
4. Update ticket assignments in ServiceNow
5. Create change log entries for audit trail

## Streamlit UI Features

- **Dashboard**: Overview of fetch, segregation, and assignment results
- **Ticket Browser**: Filter and review segregated tickets by category
- **Unknown Tickets**: Review and manually assign tickets with low confidence
- **Approval Workflow**: Approve assignments before they're sent to ServiceNow
- **Audit Log**: View change history and confidence scores
- **Manual Assignment**: Override automatic assignments for specific tickets
- **Dry-Run Mode**: Test assignments without updating ServiceNow

## Testing

Run the test suite:

```bash
# All tests
python -m pytest -q

# Specific test file
python -m pytest tests/test_servicenow_client.py -v

# With coverage
python -m pytest --cov=. tests/ -q
```

Tests include:
- API client connection and error handling
- TF-IDF segregation accuracy
- Team assignment logic
- End-to-end workflow orchestration
- Mock API response handling

## Logging

Enable detailed logging:

```python
import logging
logging.basicConfig(level=logging.DEBUG)
```

Logs will show:
- API requests and responses
- Rate limit status
- Segregation progress
- Assignment decisions
- ServiceNow update status

## Error Handling

The system handles:
- ServiceNow API unavailability (logs and continues with cached data)
- Rate limiting (automatic backoff)
- Malformed data (defaults to "Unknown" category)
- Assignment failures (tracks and retries on next run)
- Network timeouts (configurable retry logic)

## Optional: LLM Validation

To use OpenAI for validating unknown tickets:

```bash
export OPENAI_API_KEY=sk-...
```

Set `USE_LLM_VALIDATION=true` in config or Streamlit UI.

## Integration with Main Workflow

This project complements the `ticket_segregation_workflow`:

| Aspect | ticket_segregation_workflow | servicenow_api_automation |
|--------|-------|-------|
| Data source | Manual Excel upload | ServiceNow API |
| Categories | Manual Excel upload | ServiceNow API |
| Matching | TF-IDF | TF-IDF (same) |
| Output | Excel file | ServiceNow updates + Excel export |
| Assignment | Manual UI | Automated + approval workflow |
| Audit trail | CSV manual labels | ServiceNow change log |

## Next Steps

- [ ] Add retry logic for API calls
- [ ] Implement queue-based assignment for high volume
- [ ] Add Slack/email notifications for assignments
- [ ] Build historical trend dashboard
- [ ] Add custom matching rules per category
- [ ] Implement webhook for real-time updates from ServiceNow

## Support

For issues:
1. Check `.env` configuration
2. Verify ServiceNow API credentials and permissions
3. Run `python -m pytest -v` to see detailed test output
4. Check logs for API error messages

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    ServiceNow Instance                       │
│  (Incidents, Categories, Users, Assignment Groups)         │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │  ServiceNow Client   │
        │  (API, Auth, Rate    │
        │   Limiting)          │
        └──────────┬───────────┘
                   │
     ┌─────────────┴──────────────┐
     ▼                             ▼
  Tickets                      Categories
     │                             │
     └──────────────┬──────────────┘
                    ▼
       ┌────────────────────────────┐
       │ Segregation Engine         │
       │ (TF-IDF + Cosine           │
       │  Similarity)               │
       └──────────────┬─────────────┘
                      ▼
          Segregated Tickets
          (Category, Confidence)
                      │
                      ▼
       ┌────────────────────────────┐
       │ Task Assignment Engine     │
       │ (Team Mapping, Priority)   │
       └──────────────┬─────────────┘
                      ▼
          ┌────────────────────────┐
          │  Streamlit UI          │
          │  (Review & Approval)   │
          └──────────┬─────────────┘
                     ▼
          ┌────────────────────────┐
          │ ServiceNow Update      │
          │ (Assign + Audit Log)   │
          └────────────────────────┘
```
