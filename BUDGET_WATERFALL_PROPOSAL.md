# Feature Architecture & Implementation Plan: Budget Waterfall Chart

## 1. Functional & Mathematical Specification

**Goal:** Provide a visualization of how a budget balance fluctuates period-over-period (usually month-over-month) due to allocations (BudgetLimits), actual spent amounts (Transactions), and carry-over from previous periods.

**Math for the Waterfall Steps:**
For a given budget $B$, currency $C$, and period $P_i$:
- **Starting Balance ($S_i$):**
  - The carryover from the prior period. Note: Firefly III handles auto-budgets via explicitly calculated limits rather than implicit historical summing. If auto-rollover is used, the carry-over is already baked into the generated `BudgetLimit` ($A_i$). To prevent double-counting, the API needs to distinguish between simple visual accumulation (where the user has no auto-rollover and wants to see a hypothetical carryover track) vs. true `AutoBudget` setups where Firefly III has already adjusted $A_i$. The waterfall chart will display standard rollover accumulation *only* if AutoBudget rollover hasn't overridden it, or it will visualize the generated `BudgetLimit` appropriately.
- **Target Allocation / Budget Limit ($A_i$):**
  - The budgeted amount defined for period $P_i$ (`BudgetLimit` for the period). Note that this limit is tied to a specific currency.
- **Actual Spent ($V_i$):**
  - Total sum of `Transaction` amounts linked to budget $B$ in period $P_i$, matching the same currency (usually negative for expenses).
- **Net Variance ($N_i$):**
  - $N_i = A_i + V_i$ (e.g., Allocated $1000, Spent -$800 $\rightarrow$ Net = $200 under-budget).
- **Ending Balance ($E_i$):**
  - $E_i = S_i + N_i = S_i + A_i + V_i$.

**Edge Cases to Handle:**
- **Auto-rollover / Auto-adjustments:** Because Firefly III generates adjusted future limits based on past behavior (Reset, Rollover, Adjusted), the API must determine if carryover is already encoded in $A_i$ to avoid double-counting.
- **Multi-currency limits:** A budget can have multiple active `BudgetLimits` in different currencies simultaneously. The waterfall chart and API endpoint must be scoped to a single currency at a time (e.g., passing a `currency_id` to the API or iterating over them separately in the response).
- **Pro-rated / Non-monthly periods:** Firefly III supports weekly and other period budgets. The chart should scale to the `BudgetLimit` periods (weekly/monthly/yearly), relying on `BudgetRepository`'s existing pro-rating and date-handling logic rather than assuming strict calendar months.
- **Mid-period budget edits:** Changes to the limit alter $A_i$ for that specific period, which should reflect automatically since we use the stored limits.

---

## 2. Backend & Data Pipeline (Laravel)

**Existing Components to Leverage:**
- `FireflyIII\Models\Budget`: To retrieve the budget.
- `FireflyIII\Models\BudgetLimit`: To retrieve allocations per period.
- `FireflyIII\Repositories\Budget\BudgetRepository`: To fetch actual spent natively handling pro-rating logic and periods.
- `FireflyIII\Generator\Chart\Basic\ChartJsGenerator`: To format the output natively for Chart.js, maintaining consistency with other Firefly III chart endpoints.
- Caching: Use existing caching mechanisms for aggregation endpoints.

**New API Endpoint:**
`GET /api/v1/chart/budgets/{id}/waterfall`
*(Optional query params: `?start=2023-01-01&end=2023-12-31&currency_id=5`)*
The endpoint must accept a specific `currency_id` (defaulting to the user's default currency or the budget's primary limit currency) due to mixed-currency budgets.

**JSON Payload Structure (Processed via ChartJsGenerator):**
Returns structured datasets optimized for Chart.js floating bars natively:

```json
{
  "title": "Waterfall Chart for Grocery (USD)",
  "labels": ["Oct 2023", "Nov 2023"],
  "datasets": [
    {
      "label": "Waterfall Breakdown",
      "data": [
        { "period": "2023-10", "start": 0, "allocation": 1000, "spent": -850, "end": 150 },
        { "period": "2023-11", "start": 150, "allocation": 1000, "spent": -1100, "end": 50 }
      ],
      "type": "bar"
    }
  ]
}
```
*(The actual dataset configuration will leverage ChartJsGenerator arrays `[[start, start+allocation], [start+allocation, end]]` for floating bars).*

---

## 3. Frontend & Visualization

**Chart Framework:**
Firefly III uses **Chart.js**. We will utilize the **floating bar chart** functionality natively.

**UI Components Needed (Blade / AlpineJS - v2 Layout & Twig / Vue - v1 Layout):**
As Firefly III transitions from v1 to v2, the primary implementation should target the layout standards for the specific view where it will be placed (e.g., standard API consumption in JS).
1. **Budget Selector:** Multi-select to choose one or more budgets.
2. **Currency Selector:** Dropdown to switch between currencies if the budget contains mixed-currency limits.
3. **Date Range / Granularity Selector:** Uses the standard Firefly III date range pickers. Granularity should map to the budget's own frequency (weekly vs monthly).
4. **Chart Area:**
   - X-Axis: Budget Periods.
   - Y-Axis: Selected Currency amounts.
5. **Tooltips:** Custom Chart.js tooltip displaying the exact breakdown (Starting + Allocation + Spent = Ending).

**Accessibility & Dark Mode:**
- Use standard Firefly III CSS variables for colors (e.g., `--positive-color`, `--negative-color`).
- Provide a visually hidden data table backing the chart for screen readers (`<table class="sr-only">`).

---

## 4. Step-by-Step Implementation Roadmap

**Phase 1: Backend API & Calculations (PR #1)**
- Clarify exact Rollover semantics: Ensure the chart logic respects `AutoBudget` setups without double-counting carried-over limits.
- Extend `FireflyIII\Repositories\Budget\BudgetRepository` to aggregate waterfall logic.
- Add `GET /api/v1/chart/budgets/{id}/waterfall` in `FireflyIII\Api\V1\Controllers\Chart\BudgetController`. Scoped by `currency_id`.
- Implement response formatting using `FireflyIII\Generator\Chart\Basic\ChartJsGenerator`.
- **Tests:** Add Pest/PHPUnit tests. Mock standard budgets, multi-currency limits, and auto-rollover budgets.

**Phase 2: Frontend Data Fetching & Chart Component (PR #2)**
- Implement the chart on the frontend (targeting the appropriate layout—v1/v2 depending on the exact integration point in Reports).
- Configure Chart.js using floating bar arrays to render the waterfall steps based on the unified API payload.

**Phase 3: Dashboard/Report Integration (PR #3)**
- Add the chart as a dedicated section in the `Reports > Budget` view.
- Ensure translation strings are added.

---

## 5. Draft GitHub Feature Request Issue

```markdown
## Feature Request: Budget Waterfall Chart

### Problem Statement
Currently, Firefly III provides excellent insights into budget limits and expenditures per period. However, for users who want to see how their budget savings/deficits compound over time, it's difficult to visualize the period-over-period carryover and net variance at a glance.

### Proposed Solution
Implement a **Waterfall Chart** for budgets over a selected timeframe (e.g., 3, 6, 12 months). This chart will break down the flow of a budget per period (handling weekly/monthly correctly):
1. **Starting Balance** (carryover from previous periods, respecting auto-budget logic to avoid double-counting)
2. **Allocation** (the Budget Limit for the period, scoped by currency)
3. **Actual Spent** (expenses tracked against the budget in that currency)
4. **Ending Balance** (the net carryover to the next period)

This visualization will make it trivially easy to see if a budget is sustainably building a buffer or bleeding into a deficit over the year.

### Visual Mock Concept
_Using Chart.js floating bars (via existing `ChartJsGenerator`):_
- **Grey Bar:** Starting Balance
- **Green Bar (Floating):** + Allocation Limit
- **Red Bar (Floating):** - Actual Spent
- **Blue Bar (Total):** = Ending Balance (Carryover)

### Value Proposition
- Enhances the `Reports` section by providing a highly requested financial visualization.
- Helps users better understand how under-spending one period benefits them in the next.
- Seamlessly integrates with the existing Budget Limit, Multi-currency logic, and Chart.js integrations without new libraries.
```
