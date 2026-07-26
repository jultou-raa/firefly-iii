# Feature Architecture & Implementation Plan: Budget Waterfall Chart

## 1. Functional & Mathematical Specification

**Goal:** Provide a visualization of how a budget balance fluctuates month-over-month due to allocations (BudgetLimits), actual spent amounts (Transactions), and roll-over from previous periods.

**Math for the Waterfall Steps:**
For a given budget $B$ and month $M_i$:
- **Starting Balance ($S_i$):**
  - If $i = 1$ (first selected month), $S_1 = $ previous accumulated carry-over. (If the budget doesn't roll over per user settings, $S_1 = 0$).
  - For $i > 1$, $S_i = E_{i-1}$ (Ending balance of previous month).
- **Target Allocation / Budget Limit ($A_i$):**
  - The budgeted amount defined for month $M_i$ (`BudgetLimit` for the month).
- **Actual Spent ($V_i$):**
  - Total sum of `Transaction` amounts linked to budget $B$ in month $M_i$ (usually negative for expenses).
- **Net Variance ($N_i$):**
  - $N_i = A_i + V_i$ (e.g., Allocated $1000, Spent -$800 $\rightarrow$ Net = $200 under-budget).
- **Ending Balance ($E_i$):**
  - $E_i = S_i + N_i = S_i + A_i + V_i$.

**Edge Cases to Handle:**
- **Mid-year budget edits:** Firefly III treats `BudgetLimit` as a period-based record. Changes to the limit just alter $A_i$ for that specific month, keeping historical math intact.
- **Currency conversions:** Budgets are tied to a currency. `TransactionCurrency` and exchange rates in Firefly III will be used to normalize actual spent into the budget's currency.
- **Multi-budget selection:** Sum the starting balances, allocations, and actual spent across all selected budgets per month.
- **Unallocated transfers:** Transfers between expense accounts assigned to a budget count towards it. Unallocated transfers are ignored unless specifically assigned to this budget.
- **Roll-over Settings:** Some budgets do not roll over (no carryover). In this case, $S_i = 0$ always, and the chart simply reflects intra-month variance without cumulative carryover.

---

## 2. Backend & Data Pipeline (Laravel)

**Existing Components to Leverage:**
- `App\Models\Budget`: To retrieve the budget.
- `App\Models\BudgetLimit`: To retrieve allocations per month.
- `App\Repositories\Budget\BudgetRepository`: To fetch actual spent using methods like `spentInPeriod()`.
- `App\Support\Report\Budget\BudgetReportGenerator`: To fetch historical data and roll-over mechanics.

**New API Endpoint:**
`GET /api/v1/chart/budgets/{id}/waterfall`
*(Optional query params: `?start=2023-01-01&end=2023-12-31`)*

**JSON Payload Structure:**
To optimize for performance across multi-year queries, we return pre-calculated aggregated steps per period.

```json
{
  "data": {
    "budget_id": 1,
    "currency_id": 5,
    "currency_code": "USD",
    "chart_data": [
      {
        "period": "2023-10",
        "start_date": "2023-10-01",
        "end_date": "2023-10-31",
        "starting_balance": 150.00,
        "allocation": 1000.00,
        "actual_spent": -850.00,
        "net_variance": 150.00,
        "ending_balance": 300.00
      },
      {
        "period": "2023-11",
        "start_date": "2023-11-01",
        "end_date": "2023-11-30",
        "starting_balance": 300.00,
        "allocation": 1000.00,
        "actual_spent": -1100.00,
        "net_variance": -100.00,
        "ending_balance": 200.00
      }
    ]
  }
}
```

---

## 3. Frontend & Visualization

**Chart Framework:**
Firefly III uses **Chart.js**. We can utilize the **floating bar chart** functionality (passing `[start, end]` data arrays to the bar chart series) in Chart.js to build a waterfall chart natively without pulling in a new library.

**UI Components Needed:**
1. **Budget Selector:** Multi-select dropdown to choose one or more budgets.
2. **Date Range / Granularity Selector:** Pre-sets (3 Months, 6 Months, 1 Year, YTD). Granularity: Monthly.
3. **Chart Area:**
   - X-Axis: Months (e.g., Oct '23, Nov '23)
   - Y-Axis: Currency amounts.
4. **Tooltips:** Custom Chart.js tooltip displaying the exact breakdown (Starting + Allocation + Spent = Ending).

**Accessibility & Dark Mode:**
- Use CSS variables for colors (e.g., `--positive-color` for allocations, `--negative-color` for spent/deficits) to ensure compatibility with Firefly III's existing light/dark themes.
- Provide a visually hidden data table backing the chart for screen readers (`<table class="sr-only">`).

---

## 4. Step-by-Step Implementation Roadmap

**Phase 1: Backend API & Calculations (PR #1)**
- Extend `BudgetRepository` (or `BudgetReportGenerator`) to calculate carryover, limits, and spent grouped by month.
- Add `GET /api/v1/chart/budgets/{id}/waterfall` in `App\Api\V1\Controllers\Chart\BudgetController`.
- **Tests:** Add Pest/PHPUnit tests under `tests/Api/V1/Chart/BudgetControllerTest.php`. Mock transactions and limits, assert math for standard and roll-over budgets.

**Phase 2: Frontend Data Fetching & Chart Component (PR #2)**
- Add Vue/JS component or extend existing Chart.js initializers to handle the new API structure.
- Configure Chart.js using floating bar arrays to render the waterfall steps.
- **Tests:** UI interaction tests.

**Phase 3: Dashboard/Report Integration (PR #3)**
- Add the chart as a widget option on the main dashboard, or as a dedicated section in the `Reports > Budget` view.
- Ensure translation strings (`lang/en/v2-reports.php`) are added.
- **Tests:** Final E2E tests validating the complete flow.

---

## 5. Draft GitHub Feature Request Issue

```markdown
## Feature Request: Monthly Budget Waterfall Chart

### Problem Statement
Currently, Firefly III provides excellent insights into budget limits and expenditures per period. However, for users who use rolling budgets or want to see how their budget savings/deficits compound over time, it's difficult to visualize the month-over-month carryover and net variance at a glance.

### Proposed Solution
Implement a **Waterfall Chart** for budgets over a selected timeframe (e.g., 3, 6, 12 months). This chart will break down the monthly flow of a budget:
1. **Starting Balance** (carryover from previous months)
2. **Allocation** (the Budget Limit for the month)
3. **Actual Spent** (expenses tracked against the budget)
4. **Ending Balance** (the net carryover to the next month)

This visualization will make it trivially easy to see if a budget is sustainably building a buffer or bleeding into a deficit over the year.

### Visual Mock Concept
_Using Chart.js floating bars:_
- **Grey Bar:** Starting Balance
- **Green Bar (Floating):** + Allocation Limit
- **Red Bar (Floating):** - Actual Spent
- **Blue Bar (Total):** = Ending Balance (Carryover)

*(Imagine a stepped waterfall where each month's ending balance becomes the baseline for the next month's allocation).*

### Value Proposition
- Enhances the `Reports` section by providing a highly requested financial visualization.
- Helps users better understand how under-spending one month benefits them in the next (reinforcing good financial habits).
- Seamlessly integrates with the existing Budget Limit and Transaction models natively using Chart.js.
```
