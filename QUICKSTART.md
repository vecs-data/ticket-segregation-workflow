# Quick Start Guide - ServiceNow Ticket Segregation Workflow

## 5-Minute Setup

### Step 1: Navigate to Workflow Folder
```powershell
cd "c:\Users\venkatesh.yekkanti\OneDrive - Accenture\Desktop\smartops-dev\ticket_segregation_workflow"
```

### Step 2: Install Dependencies (First time only)
```powershell
pip install -r requirements.txt
```

### Step 3: Generate Sample Data
```powershell
python generate_sample_data.py
```

Output:
- `servicenow_tickets.xlsx` - 120 sample tickets
- `ticket_categories.xlsx` - 100 categories

### Step 4: Run Segregation
```powershell
python ticket_segregation.py
```

Output:
- `segregated_tickets_output.xlsx` - Results with 100+ category sheets + Unknown_101

### Step 5: Review Results
Open `segregated_tickets_output.xlsx` in Excel to see:
- Each category in separate sheet
- Matched tickets with confidence scores
- Unknown/unmatched tickets in "Unknown_101" sheet
- Summary sheet with statistics

---

## Using Your Own Data

### Replace Input Files

**servicenow_tickets.xlsx**
- Column: `Ticket_Number` (unique ID)
- Column: `Short_Description` OR `Description` (text to match)
- Optional: Type, Priority, State, Created_Date, etc.

Example structure:
```
Ticket_Number | Short_Description | Priority | State | ...
INC1000000    | Pharmacy system down | High | Open | ...
INC1000001    | Network issue | Medium | In Progress | ...
```

**ticket_categories.xlsx**
- Column: `Category_ID` (1, 2, 3, ...)
- Column: `Category_Name` (exact names to match against)
- Optional: Description, Priority_Level, Assignment_Team

Example structure:
```
Category_ID | Category_Name | Description | ...
1 | System Downtime | For system availability issues | ...
2 | Network Issues | For connectivity problems | ...
```

Then run:
```powershell
python ticket_segregation.py
```

---

## Output Explanation

### segregated_tickets_output.xlsx Structure

1. **Category Sheets** (1 per category)
   - All tickets assigned to that category
   - Columns: Ticket_Number, Description, Matched_Category, Confidence_Score, etc.

2. **Unknown_101 Sheet**
   - Tickets that didn't match any category (below confidence threshold)
   - Same columns as category sheets
   - Requires manual review or LLM validation

3. **Summary Sheet**
   - Category name, ticket count, percentage distribution
   - Sorted by ticket count (highest first)

### Confidence Score Explanation
- **0.8 - 1.0**: Excellent match (definite category)
- **0.5 - 0.8**: Good match (likely correct)
- **0.3 - 0.5**: Weak match (possibly correct)
- **< 0.3**: No match (Unknown_101 category)

---

## Advanced Usage

### Enable LLM Validation (OpenAI)

1. **Get API Key**
   - Sign up at https://openai.com
   - Get API key from account settings

2. **Create .env file**
   ```
   OPENAI_API_KEY=sk-your-key-here
   USE_LLM_VALIDATION=true
   LLM_MODEL=gpt-3.5-turbo
   ```

3. **Run with LLM**
   ```powershell
   python ticket_segregation.py
   ```
   - System will analyze unknown tickets
   - LLM provides category suggestions
   - Results appear in output Excel

### Custom Configuration

Edit `.env` file:
```ini
# Lower = stricter matching
CONFIDENCE_THRESHOLD=0.2

# Use LLM for unknown tickets
USE_LLM_VALIDATION=true

# Your OpenAI key
OPENAI_API_KEY=sk-...

# LLM settings
LLM_TEMPERATURE=0.3
```

### Utility Commands

```powershell
python workflow_utils.py  # Show all available utilities
```

Or in Python:
```python
from workflow_utils import WorkflowUtils

# Validate your input files
WorkflowUtils.validate_input_files('servicenow_tickets.xlsx', 'ticket_categories.xlsx')

# Extract unmatched tickets
unmatched = WorkflowUtils.get_unmatched_tickets('segregated_tickets_output.xlsx')
print(unmatched)

# Get statistics
stats = WorkflowUtils.export_category_statistics('segregated_tickets_output.xlsx')

# Create backup before processing
WorkflowUtils.backup_output('segregated_tickets_output.xlsx')

# Clean sensitive data
cleaned = WorkflowUtils.clean_ticket_descriptions('servicenow_tickets.xlsx', 'cleaned.xlsx')
```

---

## Common Issues & Solutions

### Issue: "ModuleNotFoundError: pandas"
**Solution:**
```powershell
pip install pandas openpyxl scikit-learn openai python-dotenv
```

### Issue: "File not found: servicenow_tickets.xlsx"
**Solution:**
```powershell
python generate_sample_data.py
```

### Issue: "OPENAI_API_KEY not found"
**Solution:**
- Create `.env` file with your API key, OR
- Set `USE_LLM_VALIDATION=false` in code to skip LLM

### Issue: Poor matching results
**Solution:**
- Lower `CONFIDENCE_THRESHOLD` (e.g., 0.2 instead of 0.3)
- Ensure category names match ticket descriptions
- Enable LLM validation for better accuracy

### Issue: Output file not created
**Solution:**
- Check if there are error messages in console
- Verify input files have correct column names
- Ensure write permissions in the folder

---

## Performance Tips

1. **For Large Datasets (1000+ tickets)**
   - Processing may take a few minutes
   - LLM validation will take longer (API calls)
   - Consider processing in batches

2. **For Better Matching**
   - Use descriptive ticket descriptions (50+ characters)
   - Ensure category names are clear and specific
   - Adjust confidence threshold based on results

3. **Memory Usage**
   - ~500MB for 1000 tickets
   - Larger categories = slower processing
   - Consider splitting large datasets

---

## Integration Examples

### Python Integration
```python
from ticket_segregation import TicketSegregationEngine

# Run segregation programmatically
engine = TicketSegregationEngine('tickets.xlsx', 'categories.xlsx')
engine.segregate_tickets(confidence_threshold=0.3)
engine.save_to_excel('output.xlsx')
report = engine.generate_report()
```

### Batch Processing
```python
import os
import glob

# Process multiple ticket files
for ticket_file in glob.glob('tickets_*.xlsx'):
    engine = TicketSegregationEngine(ticket_file, 'categories.xlsx')
    engine.segregate_tickets()
    output_name = f"output_{ticket_file}"
    engine.save_to_excel(output_name)
    print(f"Processed {ticket_file}")
```

### Scheduled Jobs (Windows Task Scheduler)
```powershell
# Create scheduled task to run weekly
$action = New-ScheduledTaskAction -Execute "C:\Users\venkatesh.yekkanti\AppData\Local\Programs\Python\Python313\python.exe" `
    -Argument "c:\path\to\ticket_segregation.py"
$trigger = New-ScheduledTaskTrigger -Weekly -DaysOfWeek Monday -At 9am
Register-ScheduledTask -Action $action -Trigger $trigger -TaskName "TicketSegregation"
```

---

## Support

For issues or questions:
1. Check troubleshooting section above
2. Review README.md for detailed documentation
3. Check console output for error messages
4. Verify input file format matches requirements

---

**Version**: 1.0  
**Last Updated**: November 2025
