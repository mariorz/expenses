# Agent Learnings

## File Structure

```
amex/
├── statements/              # Canonical monthly Amex statements (source data)
│   ├── 2025/
│   │   ├── 2025-05.csv
│   │   ├── 2025-06.csv
│   │   └── ...
│   └── 2026/
├── nubank/                  # Nubank account data
│   ├── source_pdfs/         # Original PDF statements (drop zone)
│   │   ├── credit/          # Credit card (Tarjeta) PDFs
│   │   └── debit/           # Debit account (Cuenta Nu) PDFs
│   └── statements/          # Converted CSV statements
│       ├── credit/
│       │   └── 2025/
│       │       ├── 2025-10.csv
│       │       ├── 2025-11.csv
│       │       └── 2025-12.csv
│       └── debit/
│           └── 2025/
│               ├── 2025-11.csv
│               └── 2025-12.csv
├── trips/                   # Trip expense summaries
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
Date,Description,Category,Amount_MXN,Amount_USD,Source_File,Status
```

| Column | Description |
|--------|-------------|
| Date | Transaction date (YYYY-MM-DD) |
| Description | Merchant/vendor name |
| Category | Expense category (see classifications below) |
| Amount_MXN | Amount in Mexican Pesos |
| Amount_USD | Amount in US Dollars |
| Source_File | Original monthly CSV filename for traceability |
| Status | REIMBURSED, blank, or other tracking status |

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
After processing a monthly statement, add a todo item to Roam Research for the first Saturday of the following month to remind the user to add the next Amex and Nubank statements.

Example: After processing January 2026 statements, add a Roam todo for the first Saturday in February: "Add Amex and Nubank statements for February 2026"

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
Date,Description,Category,Amount_MXN,Amount_USD,Source_File,Status
```

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

### Work Expenses (Developer Tools)
- **GITHUB, INC.** - Code repository/collaboration

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
