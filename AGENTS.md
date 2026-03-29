# Agent Learnings

## Monthly Processing Rules (HARD RULE)

Every month MUST be fully processed before moving to the next. No gaps allowed.

### Three Statements Per Month

Each month requires **3 statements** to be considered complete:

| # | Statement | Source |
|---|-----------|--------|
| 1 | **Amex** | `statements/amex/{year}/YYYY-MM.csv` |
| 2 | **Banregio** | `statements/banregio/{year}/YYYY-MM.csv` |
| 3 | **Nubank Credit** | `statements/nubank/credit/{year}/YYYY-MM.csv` |

Statements can arrive in the inbox in any order. Process each as it comes, but track the month's state:

- **Partially processed**: 1 or 2 of 3 statements done. Remind user which are missing.
- **Statements complete**: All 3 processed. Generate the monthly report (see below).
- **Fully closed**: All work expenses marked REIMBURSED. Month is done.

### After All 3 Statements Are Processed

1. **Print a monthly report** including:
   - Total spending (personal vs. work)
   - Top charges
   - Repeated merchants
   - Subscriptions/recurring charges
   - Spending by category groups
   - Any active trip breakdown (with category totals)

2. **List all non-reimbursed work expenses** for the month explicitly, and ask the user to submit them in Deel.

3. **Wait for reimbursement confirmation.** Only after the user confirms all work expenses are marked REIMBURSED (or explicitly waived) can the month be marked as fully closed.

4. **Only then** open the inbox for the next month's processing.

### Processing Order

Months must be processed sequentially. Do not start processing month N+1 until month N is fully closed (all 3 statements + reimbursements confirmed).

---

## File Structure

```
inbox/                       # Drop zone for new files to process
statements/
├── amex/                    # Canonical monthly Amex statements (source data)
│   ├── 2025/
│   │   ├── 2025-05.csv
│   │   ├── 2025-06.csv
│   │   └── ...
│   └── 2026/
├── banregio/                # Banregio checking account (debit)
│   └── 2025/
│       ├── 2025-01.csv      # All months complete (Jan-Dec)
│       └── ...
├── nubank/                  # Nubank account data
│   ├── source_pdfs/         # Original PDF statements (drop zone)
│   │   ├── credit/          # Credit card (Tarjeta) PDFs
│   │   └── debit/           # Debit account (Cuenta Nu) PDFs
│   └── credit/              # Converted CSV statements
│   │   └── 2025/
│   │       ├── 2025-10.csv
│   │       ├── 2025-11.csv
│   │       └── 2025-12.csv
│   └── debit/
│       └── 2025/
│           ├── 2025-11.csv
│           └── 2025-12.csv
consolidated/                # Combined expenses from all sources
│   ├── 2025.csv             # All 2025 expenses in unified schema
│   └── 2026.csv
trips/                       # Trip expense summaries
│   ├── 2025/
│   │   ├── argentina_oct-nov.csv
│   │   ├── merida_dec.csv
│   │   ├── nyc_dec27-jan1.csv
│   │   └── yucatan_jul.csv
│   └── 2026/
├── work_expenses/           # Work expenses for reimbursement tracking
│   ├── 2025.csv
│   └── 2026.csv
├── AGENTS.md                # This file - agent instructions
└── CLAUDE.md                # Project instructions
```

### Naming Conventions
- **Statements**: `YYYY-MM.csv` (e.g., `2025-01.csv` for January 2025)
- **Trips**: `{destination}_{month(s)}.csv` organized by year folder
- **Work Expenses**: `{year}.csv` (one file per year)

### Column Schema (for derived files)

Both `work_expenses/{year}.csv` and `trips/{year}/*.csv` use this unified schema:

```
Date,Description,Category,Amount_MXN,Amount_USD,Source_File,Account_Type,Status
```

| Column | Description |
|--------|-------------|
| Date | Transaction date (YYYY-MM-DD) |
| Description | Merchant/vendor name |
| Category | Expense category (see classifications below) |
| Amount_MXN | Amount in Mexican Pesos |
| Amount_USD | Amount in US Dollars |
| Source_File | Original monthly CSV filename for traceability |
| Account_Type | `credit` (Amex, Nubank credit) or `debit` (Nubank debit) |
| Status | REIMBURSED, blank, or other tracking status |

### Account Type Determination

| Source_File Pattern | Account_Type |
|---------------------|--------------|
| `YYYY-MM.csv` (Amex) | credit |
| `nubank-credit-*.csv` | credit |
| `nubank-debit-*.csv` | debit |
| `banregio-*.pdf` | debit |

### Transactions to Exclude from Expense Totals

The following transaction types are **NOT expenses** and should be excluded when calculating spending totals:

| Category | Examples | Reason |
|----------|----------|--------|
| Internal transfers | Nubank-to-Nubank moves, "Límite convertido en saldo" | Moving money between own accounts |
| Credit card payments | "GRACIAS POR SU PAGO", "Pago a tu tarjeta de crédito" | Paying off credit liability, not new spending |

**What IS included as Personal expenses:**
- **Transfer:Karen** - Reimbursements to partner for shared costs she paid
- **Transfer:Luci** - Household staff payments
- **Housing:Mortgage** - Monthly mortgage payments
- **ATM withdrawals** - Cash spending (even if individual purchases untracked)

---

## Processing the Inbox

The `inbox/` directory is the drop zone for new files. When the user adds files here and asks to process them, follow this workflow.

### Step 1: List Inbox Contents

```bash
ls -la inbox/
```

### Step 2: Identify File Types and Route Accordingly

| File Type | Identification | Action |
|-----------|----------------|--------|
| **ZIP** | `.zip` extension | Extract to temp dir, then process contents |
| **Banregio PDF** | `MARIO ROMERO ZAVALA` or `85020113` in filename | Process per "Processing Banregio PDF Statements" section |
| **Nubank PDF** | `estado de cuenta` or Nubank identifiers | Process per "Processing Nubank PDF Statements" section |
| **Amex CSV** | Contains `Fecha,Descripción,Importe` header | Move to `statements/amex/{year}/YYYY-MM.csv` |
| **Nubank CSV** | Contains unified schema + `nubank` in name | Move to `statements/nubank/{credit\|debit}/{year}/` |

### Step 3: Process Each File

**For ZIPs:**
```bash
unzip inbox/filename.zip -d /tmp/inbox_extract/
```
Then process each extracted file according to its type.

**For Banregio PDFs:**
1. Read page 1 to determine statement month
2. Extract Transfer:Karen, Transfer:Luci, Housing:Mortgage
3. Create CSV in `statements/banregio/{year}/YYYY-MM.csv`
4. Move PDF to `statements/banregio/source_pdfs/banregio-YYYY-MM.pdf`

**For Nubank PDFs:**
1. Classify as credit or debit (see Nubank section)
2. Extract transactions
3. Create CSV in `statements/nubank/{credit|debit}/{year}/YYYY-MM.csv`
4. Move PDF to `statements/nubank/source_pdfs/{credit|debit}/`

**For Amex CSVs:**
1. Check header row for `Fecha` to confirm format
2. Determine month from transactions (most common month in file)
3. Move to `statements/amex/{year}/YYYY-MM.csv`

### Step 4: Clean Up

After processing each file:
1. Delete from inbox (or move to source_pdfs for PDFs)
2. Regenerate consolidated file if statements were added
3. Report what was processed

### Example Inbox Processing Session

```
User: process inbox

Agent:
1. Found: banregio-statement.pdf, nubank-nov.pdf, 2025-12.csv
2. Processing banregio-statement.pdf...
   - Identified as December 2025
   - Created statements/banregio/2025/2025-12.csv
   - Moved PDF to statements/banregio/source_pdfs/banregio-2025-12.pdf
3. Processing nubank-nov.pdf...
   - Identified as Credit, November 2025
   - Created statements/nubank/credit/2025/2025-11.csv
   - Moved PDF to statements/nubank/source_pdfs/credit/
4. Processing 2025-12.csv...
   - Identified as Amex, December 2025
   - Moved to statements/amex/2025/2025-12.csv
5. Regenerating consolidated/2025.csv...
6. Done. Inbox is empty.
```

---

## Processing New Monthly CSV Files

When a new monthly CSV is added (e.g., `statements/2026/2026-01.csv`), follow these steps:

### Step 1: Check Watchlist
Search the new CSV for all patterns in the Watchlist section below. Flag any matches for user review.

### Step 2: Identify Work Expenses
Search for known work expense vendors (see classifications below) and add new entries to `work_expenses/{year}.csv` with:
- Appropriate category
- USD conversion (use current exchange rate if not provided)
- Source_File pointing to the new CSV
- Status left blank (user marks as REIMBURSED later)

### Step 3: Check for Trip-Related Expenses
If the user mentions a trip, or if charges match known trip patterns:
- Create a new file in `trips/{year}/` named `{destination}_{month(s)}.csv`
- Include all related charges with categories: Flights, Accommodation, Transportation, Food, Shopping, Entertainment, Services, Refund

### Step 4: Currency Conversion
- If USD amount is missing, calculate using current MXN/USD exchange rate
- Note: Amex CSVs sometimes include USD in format like `"1,150.04 USD"` or as separate column

### Step 5: Add Roam Reminder for Next Month
After processing month N's statement, add a todo item to Roam Research for the first Saturday of month N+2 to remind the user to add month N+1's statements (which won't be available until that month ends).

Example: After processing January 2026 statements, add a Roam todo for the first Saturday in **March**: "Add Amex and Nubank statements for February 2026"

### Step 6: Regenerate Consolidated File
After processing any new statements, regenerate the consolidated expenses file for that year (see "Generating Consolidated Expenses" section below).

---

## Generating Consolidated Expenses

The `consolidated/{year}.csv` file combines ALL expenses from all sources into a single file with the unified schema. This is the master file for expense analysis.

### Sources to Include

| Source | Path | Format |
|--------|------|--------|
| Amex | `statements/amex/{year}/*.csv` | Raw (needs conversion) |
| Nubank Credit | `statements/nubank/credit/{year}/*.csv` | Unified schema |
| Nubank Debit | `statements/nubank/debit/{year}/*.csv` | Unified schema |
| Banregio | `statements/banregio/{year}/*.csv` | Unified schema |

### Amex Format Conversion

Amex CSVs use Spanish column names and need conversion:

| Amex Column | Unified Column |
|-------------|----------------|
| Fecha | Date (convert "22 Dec 2025" → "2025-12-22") |
| Descripción | Description |
| Importe | Amount_MXN |
| Monto en moneda extranjera | Amount_USD (extract number from "250.00 USD") |
| (derived) | Source_File = "{year}-{MM}.csv" |
| (fixed) | Account_Type = "credit" |
| (empty) | Category, Status |

### Output Schema

```
Date,Description,Category,Amount_MXN,Amount_USD,Source_File,Account_Type,Status
```

### Generation Rules

1. **Sort by Date** - All transactions sorted chronologically (oldest first)
2. **Preserve Source_File** - Keep original source file reference for traceability
3. **No Deduplication** - Same charge can appear in multiple sources (e.g., if manually added to trips); keep all
4. **Include All Categories** - Including Transfer:Karen, Transfer:Luci, Housing:Mortgage from Banregio

### When to Regenerate

Regenerate the consolidated file when:
- New monthly statements are added
- Existing CSVs are corrected/updated
- New source types are added

---

## Generating Monthly Spend Reports

Use the consolidated file to generate monthly spending summaries.

### Expense Categories

| Category | Includes | Type |
|----------|----------|------|
| **Personal** | All spending + Mortgage + Transfers (Karen/Luci) + ATM | Personal |
| **Work** | Anthropic, OpenAI, Claude.AI, AWS, Google Cloud, GitHub, Discord, Bankless, Ankr | Reimbursable |

### Work Expense Patterns

Search for these patterns to identify work expenses:
```
ANTHROPIC, OPENAI, CHATGPT, CLAUDE.AI, CLAUDE AI,
AWS EMEA, AWS , AMAZON WEB SERVICES,
GOOGLE CLOUD, PAYU-GOOGLE,
ANKR, BANKLESS, DISCORD,
ENVIO
```

### Transactions to EXCLUDE from Totals

- **Credit card payments**: "GRACIAS POR SU PAGO" (these are payments TO the card, not spending)
- **Negative amounts**: Refunds should offset purchases, but large refunds may need review

### Monthly Report Format

```
Month        Personal         Work
--------------------------------------
2025-01      XXX,XXX.XX   XXX,XXX.XX
2025-02      XXX,XXX.XX   XXX,XXX.XX
...
--------------------------------------
TOTAL      X,XXX,XXX.XX   XXX,XXX.XX
```

### 2025 Monthly Breakdown (MXN)

| Month | Personal    | Work       | Total       |
|-------|-------------|------------|-------------|
| Jan   | 233,263     | 1,286      | 234,549     |
| Feb   | 232,133     | 6,012      | 238,145     |
| Mar   | 186,926     | 6,187      | 193,113     |
| Apr   | 311,004     | 8,637      | 319,641     |
| May   | 177,553     | 12,750     | 190,302     |
| Jun   | 151,760     | 126,253    | 278,013     |
| Jul   | 250,062     | 14,100     | 264,162     |
| Aug   | 188,354     | 166,532    | 354,886     |
| Sep   | 236,083     | 18,807     | 254,890     |
| Oct   | 370,198     | 8,775      | 378,972     |
| Nov   | 278,189     | 150,676    | 428,865     |
| Dec   | 313,799     | 8,367      | 322,167     |
|-------|-------------|------------|-------------|
| **Total** | **2,929,324** | **528,382** | **3,457,705** |

### Personal Spending Components

Personal spending includes ALL of these (do not separate):
- **Regular spending**: Credit card purchases, debit purchases
- **Housing:Mortgage**: Monthly mortgage payments from Banregio
- **Transfer:Karen**: Reimbursements to partner for shared costs
- **Transfer:Luci**: Household staff payments
- **ATM**: Cash withdrawals (represents untracked cash spending)

### Sample Python Script

```python
import csv
from collections import defaultdict

work_patterns = [
    'ANTHROPIC', 'OPENAI', 'CHATGPT', 'CLAUDE.AI', 'CLAUDE AI',
    'AWS EMEA', 'AWS ', 'AMAZON WEB SERVICES',
    'GOOGLE CLOUD', 'PAYU-GOOGLE', 'ANKR', 'BANKLESS', 'GITHUB', 'DISCORD'
]

def is_work_expense(desc):
    return any(p in desc.upper() for p in work_patterns)

personal_by_month = defaultdict(float)
work_by_month = defaultdict(float)

with open('consolidated/2025.csv', 'r') as f:
    reader = csv.DictReader(f)
    for row in reader:
        date, desc = row['Date'], row['Description']
        if not date or len(date) < 7:
            continue
        month = date[:7]

        amount = float(row['Amount_MXN'].replace(',', '') or 0)

        # Skip credit card payments
        if 'GRACIAS POR SU PAGO' in desc.upper():
            continue

        # Categorize
        if is_work_expense(desc) and amount > 0:
            work_by_month[month] += amount
        else:
            personal_by_month[month] += amount

# Print report
for month in sorted(personal_by_month.keys()):
    print(f"{month}  {personal_by_month[month]:>12,.2f}  {work_by_month[month]:>12,.2f}")
```

---

## Processing Nubank PDF Statements

When new Nubank PDFs are dropped into `nubank/source_pdfs/`, follow these steps:

### Step 1: Classify PDF Type

Read each PDF and identify the account type:

| Type | Identifiers | Period |
|------|-------------|--------|
| **Credit (Tarjeta)** | "TARJETA:", "Límite de crédito", "Saldo total del periodo" | 22nd to 22nd (e.g., "22 SEP 2025 - 22 OCT 2025") |
| **Debit (Cuenta Nu)** | "Cuenta Nu:", "CLABE:", "Saldo inicial" | 1st to end of month (e.g., "01 al 30 nov 2025") |

### Step 2: Move PDF to Correct Folder

- Credit card PDFs → `nubank/source_pdfs/credit/`
- Debit account PDFs → `nubank/source_pdfs/debit/`

### Step 3: Extract Transactions

Parse the "TRANSACCIONES" or "Detalle de movimientos" section. Skip:
- Payments ("Pago a tu tarjeta de crédito")
- Internal adjustments ("Ajuste", "Abono por plan de pagos fijos")
- Fees/interest (unless tracking separately)
- "Límite convertido en saldo" (credit-to-debit internal transfer)

### Step 4: Create CSV

Save to `nubank/statements/{credit|debit}/YYYY/YYYY-MM.csv` using the statement's closing month.

**CSV Schema** (same as Amex-derived files):
```
Date,Description,Category,Amount_MXN,Amount_USD,Source_File,Account_Type,Status
```

Set `Account_Type` to `credit` for Tarjeta statements, `debit` for Cuenta Nu statements.

**Categories from Nubank PDFs:**
- Restaurante, Hogar, Otros, Electrónicos, Supermercado, Ocio, Transportation, Transfer, ATM

**Notes:**
- Add "(Argentina)" suffix for ARS transactions during trips
- "Of" entries are likely OnlyFans subscriptions (category: Otros)
- Include USD amount when foreign currency conversion is shown

### Step 5: Add Roam Reminder for Next Month
Same as Amex processing - add a todo to Roam Research for the first Saturday of the following month.

### Nubank Statement Naming

| PDF Title | Account | Period | CSV Name |
|-----------|---------|--------|----------|
| "estado de cuenta de octubre" | Credit | 22 Sep - 22 Oct | 2025-10.csv |
| "estado de cuenta de Noviembre" | Debit | 01-30 Nov | 2025-11.csv |

---

## Expense Classification

### NOT Work Expenses
- **Starlink** - Personal internet service
- **X Corp Paid Features** - Personal social media
- **YouTube Premium** - Personal entertainment
- **Google Nest** - Personal smart home
- **Whitepaper.mx** - Personal
- **GITHUB, INC.** - Personal (confirmed 2026-03-29)

### Known Merchants (unclear names)
- **WP100** - Wolf gym (Pachuca 100, Condesa) - Fitness membership

### Work Expenses (AI Services)
- **Anthropic** - AI/LLM service
- **OpenAI / CHATGPT SUBSCR** - AI/LLM service
- **CLAUDE.AI SUBSCRIPTION** - AI/LLM service

### Work Expenses (Cloud/Infrastructure)
- **AWS EMEA** - Cloud infrastructure
- **PAYU-GOOGLE CLOUD** - Cloud services
- **ANKR PBC** - Blockchain infrastructure

### Work Expenses (Education/Research)
- **Bankless** - Web3/blockchain industry research

### Work Expenses (Cloud/Infrastructure continued)
- **ENVIO** - Cloud/infrastructure service (London-based)

### Work Expenses (Communication)
- **Discord Nitro** - Work communication (annual subscription)

### Work Expenses (Meals)
- Business meals (requires user confirmation)

### Potentially Work (Needs Confirmation)
- **Google One** - Storage (could be personal or work)

---

## Watchlist - Flag When Found

When processing new monthly CSV files, search for and flag these vendors:

| Vendor | Search Pattern | Reason | First Seen |
|--------|---------------|--------|------------|
| Mariemur | `MARIEMUR` | Monitor for unexpected new charges | Dec 2025 (378.32 + 7,245.88 MXN expected) |
| NYC Trip - Hotel | See hotel patterns below | Check for extra hotel charges (Dec 27 - Jan 1 trip) | Dec 2025 (64,428.62 MXN / 3,449.62 USD booked via Travelocity) |
| NYC Trip - In-trip expenses | See hotel + location patterns | **January CSV will have actual trip expenses** (Uber, meals, etc.) | Trip: Dec 27, 2025 - Jan 1, 2026 |

### Hotel/Accommodation Search Patterns

When searching for hotel-related charges, use these inclusive patterns:

**Booking platforms:**
```
TRAVELOCITY|EXPEDIA|BOOKING\.COM|HOTELS\.COM|AIRBNB|VRBO|KAYAK|HOTWIRE|PRICELINE|AGODA
```

**Major hotel chains:**
```
MARRIOTT|HILTON|HYATT|IHG|WYNDHAM|BEST WESTERN|RADISSON|SHERATON|WESTIN|RITZ|FOUR SEASONS|HOLIDAY INN|HAMPTON|DOUBLETREE|EMBASSY SUITES|COURTYARD|FAIRFIELD|RESIDENCE INN|SPRINGHILL|ALOFT|W HOTEL|ST REGIS|WALDORF|CONRAD
```

**Generic hotel terms:**
```
HOTEL|MOTEL|INN|RESORT|SUITES|LODGING|ACCOMMODATION
```

**Location-specific (for NYC trip):**
```
NEW YORK|NYC|MANHATTAN|BROOKLYN|QUEENS
```

**Combined search for NYC hotel extras:**
```
(HOTEL|MARRIOTT|HILTON|HYATT|INN|RESORT).*(NEW YORK|NYC|MANHATTAN)|(NEW YORK|NYC|MANHATTAN).*(HOTEL|ROOM|STAY)
```

> **Note:** Hotel incidentals (minibar, room service, parking) often appear as separate charges under the hotel's name, not the booking platform.

---

## Trip History

| Trip | Dates | File | Total (MXN) | Total (USD) | Notes |
|------|-------|------|-------------|-------------|-------|
| Yucatan | Jul 2025 | `trips/2025/yucatan_jul.csv` | ~14,944 | ~840 | Short trip; dates TBD |
| Argentina | Oct 26 - Nov 22, 2025 | `trips/2025/argentina_oct-nov.csv` | ~285,600 | ~16,050 | Complete (Amex + Nubank) |
| Merida | Dec 14-21, 2025 | `trips/2025/merida_dec.csv` | ~27,645 | ~1,553 | Complete |
| NYC | Dec 27, 2025 - Jan 1, 2026 | `trips/2025/nyc_dec27-jan1.csv` | ~125,119 | ~6,720 | Pre-trip + ATM; more in-trip expenses expected in 2026-01.csv |

---

## Deel Reimbursement Reconciliation (March 2026)

### AI Services Reconciliation (2025 + 2026 Q1)

**Completed 2026-03-29.** Full reconciliation of OpenAI/ChatGPT, Claude.AI, and Anthropic charges on Amex vs Deel reimbursements.

**Amex total (AI services):** ~116,565 MXN (~$6,048 USD)
- OpenAI/ChatGPT: 55,130 MXN (~$2,843 USD) — 16 charges (Jan '25 - Mar '26)
- Claude.AI Subscription: 3,595 MXN (~$180 USD) — 9 charges (Jan - Sep '25)
- Anthropic credits: 57,840 MXN (~$3,025 USD) — 18 charges (Feb - Dec '25)

**Deel total (AI services):** ~$5,663 USD
- Recurring "openai/chatgpt": $4,313 across 20 semimonthly cycles (Jun '25 - Mar '26)
- One-off "openai may,june,july": $600
- Anthropic (3 submissions): $750

**Gap: ~$385 USD** — submitted to Deel on 2026-03-29 (adjustment ID: `3a8b29b2-d707-4778-aae2-5c6683d07198`)

### Google Cloud Platform Reconciliation

PAYU-GOOGLE CLOUD charges on Amex (Apr 2025 - Mar 2026): 25 charges totaling ~4,397 MXN (~$231 USD). Never previously submitted to Deel.

**Adjustment:** $231 USD submitted 2026-03-29 (Deel ID: `bdc52a49-7129-4666-a7d5-3dec255c5e60`)

### Deel Recurring Setup (Fixed 2026-03-29)

Old recurring (REMOVED):
- "google workspace" $85 USD/cycle semimonthly — covered off-Amex Google Workspace billing
- "openai/chatgpt" 219.39 MXN/cycle semimonthly — was being paid as ~$219 USD due to currency bug

New recurring (ADDED):
- **"OpenAI subs"** $200 USD monthly — matches OPENAI *CHATGPT SUBSCR on Amex (~4th of each month)
- **"Google Cloud"** $22 USD monthly — matches PAYU-GOOGLE CLOUD on Amex (~$11 × 2/month)

Note: Google Workspace ($85) was removed because the subscription moved off the Amex after Mar 2025 and no longer needs expense reimbursement.

### Resolved Issues

1. **Recurring "openai/chatgpt" MXN/USD bug** — FIXED. Old 219.39 MXN recurring was being paid as ~$219 USD. Removed and replaced with correct $200 USD monthly.
2. **Semimonthly frequency** — FIXED. New recurring items are monthly.
3. **Anthropic charges mostly unsubmitted** — CLOSED. The $385 AI services adjustment covers the historical gap. Going forward, submit Anthropic charges individually.
4. **Google Cloud never submitted** — CLOSED. $231 backlog adjustment submitted. New $22/month recurring covers future charges.
5. **GitHub reclassified as personal** — not a work expense, removed from work expense patterns.

### AWS Reconciliation (2025)

- Jun '25 AWS ($5,877): Submitted to Deel as $5,853 ✓
- Aug '25 AWS ($7,963): Submitted to Deel as $7,843 ✓
- Aug '25 AWS: **Double submission** of $7,843 (same invoice) — settled by NOT submitting Nov '25 AWS ($7,685). Net difference: ~$158 in user's favor.

---

## Unresolved Items (Need User Clarification)

### August 2025 Vivaaerobus Charges
- Aug 3: 8,277.77 MXN
- Aug 8: 16,677.27 MXN + 1,559.53 MXN
- **Total: 26,514.57 MXN**
- **Missed flights** - booked but not taken, no trip to track

### Burning Man 2025
- Apr 30: BURNINGMAN ticket 33,731.13 MXN (in 2025-05.csv)
- **Did not attend** - tickets only, no trip to track

---

## Visualization Guidelines (Tufte Principles)

When generating charts, reports, or visual summaries, follow Edward Tufte's principles for data visualization:

### Core Principles

1. **Maximize data-ink ratio** - Every drop of ink should convey information. Remove chartjunk, unnecessary gridlines, decorative elements, and redundant labels.

2. **No pie charts for 3+ categories** - Pie charts are only acceptable for showing two values (binary comparisons). For three or more categories, use horizontal bar charts sorted by value.

3. **Small multiples over complex charts** - When comparing across time or categories, use repeated small charts with consistent scales rather than cramming everything into one busy visualization.

4. **Direct labeling** - Label data points directly rather than using legends that force the eye to travel back and forth.

5. **High information density** - Pack information densely but clearly. Tables often convey more information than charts.

### Chart Type Selection

| Data Type | Recommended | Avoid |
|-----------|-------------|-------|
| Part-to-whole (2 values) | Simple bar or binary segment | - |
| Part-to-whole (3+ values) | Horizontal bar chart, sorted | Pie chart, donut chart |
| Trends over time | Line chart, sparkline | Area chart (obscures data) |
| Comparisons | Horizontal bar chart | Vertical bars (harder to read labels) |
| Distributions | Strip plot, histogram | Box plots (hide data) |
| Correlations | Scatter plot | Bubble charts (area misleads) |

### Specific Rules

- **Horizontal bars > vertical bars** - Category labels are easier to read
- **Sort bar charts by value** - Not alphabetically (unless order has meaning)
- **Start axes at zero** for bars - Truncated axes mislead
- **Use consistent scales** across small multiples
- **Muted colors** - Avoid bright, saturated colors; use grayscale with one accent
- **No 3D effects** - Ever
- **No shadows or gradients** - Flat, clean fills only

### The Expense Explorer

The `explorer.html` file implements these principles:
- Horizontal bar charts for category and merchant breakdowns
- Binary segment chart only for two-value reimbursement status
- Sparkline-style trend chart for monthly spending
- High data-ink ratio with minimal decoration
- Tabular data for detailed exploration
- Monochrome palette with functional color only (red for watchlist alerts)

---

## Processing Banregio PDF Statements

Banregio statements are monthly "ESTADO DE CUENTA UNICO" PDFs containing checking account transactions and mortgage information.

### Step 1: Identify Statement Month

The statement period is shown on page 1:
- "del 01 al 31 de DICIEMBRE 2025" → December 2025
- "del 01 al 30 de NOVIEMBRE 2025" → November 2025

### Step 2: Extract Relevant Transactions

Only extract these three categories:

| Category | How to Identify | Search Pattern |
|----------|-----------------|----------------|
| **Transfer:Karen** | BBVA MEXICO + account 012180015490204623 | `BBVA MEXICO.*012180015490204623` |
| **Transfer:Luci** | AZTECA + account 5263540114400399 | `AZTECA.*5263540114400399` |
| **Housing:Mortgage** | "Monto Pago Actual" on mortgage summary page | Look on pages 5-7 for mortgage section |

### Step 3: Transactions to EXCLUDE

Skip these when processing (they are not household expenses):
- **Nubank transfers** - `NU MEXICO` or `nubank` in description
- **AMEX payments** - `AMERICAN EXPRESS` in description
- **PAGO REFERENCIADO SA** - Service payments (already tracked elsewhere)
- **DLOCAL / STP deposits** - Income, not expenses
- **Commissions/IVA** - Bank fees (COM. SPEI, IVA SPEI)

### Step 4: Find Mortgage Amount

The mortgage payment is on the mortgage summary page (usually pages 5-7):
- Look for "HIPOTECA VERTICAL" section
- Find "Monto Pago Actual" row → this is the total mortgage payment
- The payment date is typically the 18th-20th of the month

### Step 5: Create CSV

Save to `statements/banregio/YYYY/YYYY-MM.csv`

**CSV Schema:**
```
Date,Description,Category,Amount_MXN,Amount_USD,Source_File,Account_Type,Status
```

- Set `Account_Type` to `debit` (Banregio is a checking account)
- Leave `Amount_USD` empty
- `Source_File` = `banregio-YYYY-MM.pdf`

### Example Extraction

From December 2025 PDF:
```
2025-12-18,Transfer to Karen,Transfer:Karen,8000.00,,banregio-2025-12.pdf,debit,
2025-12-18,Mortgage Payment,Housing:Mortgage,34842.03,,banregio-2025-12.pdf,debit,
2025-12-26,Transfer to Luci,Transfer:Luci,2800.00,,banregio-2025-12.pdf,debit,
```

### Banregio Processing Status (2025)

| Month | Status | Notes |
|-------|--------|-------|
| Jan-Dec | ✓ Done | All 12 months complete |

### Banregio PDF Processing Queue

PDFs are extracted to `/tmp/banregio_all/` in numbered folders (0-11). The folder numbers do NOT correspond to months - you must read page 1 of each PDF to determine the month.

**Workflow (one PDF per session):**

1. **Get next folder:**
   ```bash
   ls /tmp/banregio_all/ | head -1
   ```

2. **Find PDF in that folder:**
   ```bash
   ls /tmp/banregio_all/{folder}/*.pdf
   ```

3. **Read ONLY page 1** to get the statement month:
   - Look for "del 01 al XX de {MONTH} {YEAR}"

4. **Check if CSV exists:**
   - If `statements/banregio/2025/2025-{MM}.csv` exists → skip to step 6
   - If not → process the full PDF (step 5)

5. **Process PDF:**
   - Extract Karen transfers (BBVA MEXICO + 012180015490204623)
   - Extract Luci transfers (AZTECA + 5263540114400399)
   - Extract mortgage payment ("Monto Pago Actual" on pages 5-7)
   - Create CSV in `statements/banregio/2025/`

6. **Delete the processed folder:**
   ```bash
   rm -rf /tmp/banregio_all/{folder}
   ```

7. **Compact conversation** - this is critical to avoid context overflow

**Repeat until `/tmp/banregio_all/` is empty.**

---

## Tools

### Expense Explorer (`explorer.html`)

A single-file HTML interface for exploring expense data. Open in any browser.

**Features:**
- Load multiple CSV files (Amex statements, Nubank, trips, work expenses)
- Unified search across all transactions
- Filter by date range, category, source, amount
- View tabs: All Transactions, Work Expenses, Trips, Monthly
- Visual summaries following Tufte principles
- Watchlist highlighting (red rows for flagged vendors)
- Export filtered data to CSV
- Keyboard shortcuts: Ctrl/Cmd+F to focus search

**Loading Data:**
1. Open `explorer.html` in a browser
2. Click "Choose Files" and select CSV files
3. Multi-select with Ctrl/Cmd+click to load several at once

**Supported CSV Formats:**
- Amex raw statements (Fecha, Descripción, Importe columns)
- Derived files (Date, Description, Category, Amount_MXN, Amount_USD, Source_File, Status)

---

## Roam Research Formatting Rules

### Email Drafts
When adding email drafts to Roam (e.g., for watchlist follow-ups), structure them as:
- **To:** (level 2 block)
- **Subject:** (level 2 block)
- **Body:** (level 2 block)
  - The entire email body as **one single block** of text (level 3), using `\n\n` for paragraph breaks within the block. Do NOT split paragraphs into separate child blocks.
