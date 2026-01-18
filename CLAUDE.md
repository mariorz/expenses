# Expense Review Workspace

This workspace contains credit card statement CSV files for expense tracking and categorization.

## Directory Structure
```
statements/
├── amex/           # American Express statements
│   └── {year}/
├── nubank/         # Nubank statements
│   ├── credit/
│   │   └── {year}/
│   ├── debit/
│   │   └── {year}/
│   └── source_pdfs/
trips/              # Trip-specific expense breakdowns
work_expenses/      # Work-related expenses for reimbursement
```

## Files
- `statements/**/*.csv` - Monthly card statements by provider and year

## Related Documentation
- [AGENTS.md](./AGENTS.md) - Learnings about expense classification for AI agents
