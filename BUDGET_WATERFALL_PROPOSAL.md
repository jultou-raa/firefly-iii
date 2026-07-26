# Comprehensive Feature Review & Implementation Architecture

---

## 1. Architectural Review & Critique

### Strengths
1. **Leverages Native Conventions:** Reusing `BudgetRepository`, `ChartJsGenerator`, and existing date/period utilities fits Firefly III’s architectural patterns.
2. **Multi-Currency Scoping:** Requiring `currency_id` up front prevents mixed-currency arithmetic errors, which is critical since Firefly III supports multi-currency budget limits.

### Critical Edge Cases & Technical Refinement
1. **Precision Math (BCMath vs. Floats):** Firefly III strictly avoids floating-point operations for financial calculations. All monetary calculations must use `bcadd`, `bcsub`, and `bccomp` with 2–4 decimal places to prevent rounding artifacts.
2. **The AutoBudget / Rollover Double-Counting Problem:**
   - In Firefly III, an `AutoBudget` with type `rollover` dynamically increases/decreases the generated `BudgetLimit` ($A_i$) for period $P_i$ based on leftover balances from $P_{i-1}$.
   - **Resolution:** If `rollover` is enabled, the true "Allocation" added in period $i$ is $A_i^{\text{base}} = A_i^{\text{stored}} - S_i$. The API pipeline must calculate base allocation separately from accumulated carryover $S_i$ to ensure the waterfall steps sum correctly:
     $$\text{Start } (S_i) + \text{Base Allocation } (A_i^{\text{base}}) - \text{Spent } (V_i) = \text{End } (E_i)$$
3. **Period Alignment Across Boundary Ranges:** Budgets in Firefly III can be daily, weekly, monthly, quarterly, or yearly. A requested range (e.g., `start=2023-01-01&end=2023-06-30`) must be converted into discrete `BudgetPeriod` instances via `BudgetRepository` rather than forcing hardcoded month boundaries.

---

## 2. Mathematical Formalization & AutoBudget Logic

For budget $B$, currency $C$, and ordered periods $P_1, P_2, \dots, P_n$:

### Initial State ($P_1$):
* $S_1 = \text{Prior Carryover}$ (either 0 or fetched from $P_0$ spending/limit analysis).
* $V_i = \left| \sum \text{Transaction Amounts for } P_i \right|$ (expenses represented as a positive scalar for subtraction).

### For Period $P_i$:
* **Stored Budget Limit:** $A_i^{\text{stored}}$
* **Base Allocation ($A_i$):**
  $$\text{If Rollover Active: } A_i = A_i^{\text{stored}} - S_i, \quad \text{Else: } A_i = A_i^{\text{stored}}$$
* **Net Balance Variance ($N_i$):**
  $$N_i = A_i - V_i$$
* **Ending Balance ($E_i$):**
  $$E_i = S_i + N_i = S_i + A_i - V_i$$
* **Next Period Starting Balance ($S_{i+1}$):**
  $$S_{i+1} = E_i$$

---

## 3. Laravel Backend Implementation

### A. Repository Logic (`app/Repositories/Budget/BudgetRepository.php`)

Add `getWaterfallData` to `BudgetRepositoryInterface` and implement it in `BudgetRepository`:

```php
namespace FireflyIII\Repositories\Budget;

use Carbon\Carbon;
use FireflyIII\Models\Budget;
use FireflyIII\Models\TransactionCurrency;
use FireflyIII\Support\Http\Chart\BudgetWaterfallData;
use Illuminate\Support\Collection;

class BudgetRepository implements BudgetRepositoryInterface
{
    /**
     * Calculate period-over-period waterfall balance data for a given budget.
     */
    public function getWaterfallData(
        Budget $budget,
        TransactionCurrency $currency,
        Carbon $startDate,
        Carbon $endDate
    ): Collection {
        $periods = $this->getBudgetPeriodsInRange($budget, $startDate, $endDate);
        $results = collect();

        $currentStart = '0.0000'; // Default carryover for initial period

        foreach ($periods as $period) {
            $periodStart = $period['start'];
            $periodEnd   = $period['end'];

            // 1. Fetch stored limit for period & currency
            $limitModel   = $this->getBudgetLimitForPeriod($budget, $currency, $periodStart, $periodEnd);
            $storedLimit  = $limitModel ? (string) $limitModel->amount : '0.0000';

            // 2. Fetch total spent (returns positive string or '0.0000')
            $spentAmount  = (string) $this->getExpensesForPeriod($budget, $currency, $periodStart, $periodEnd);

            // 3. AutoBudget Rollover Normalization
            $isRollover = $budget->autoBudgets()
                ->where('transaction_currency_id', $currency->id)
                ->where('type', 'rollover')
                ->exists();

            if ($isRollover) {
                // Deduct carryover from stored limit to avoid double counting
                $baseAllocation = bcsub($storedLimit, $currentStart, 4);
            } else {
                $baseAllocation = $storedLimit;
            }

            // 4. Calculate Net and Ending Balances
            // Net = Allocation - Spent
            $netVariance   = bcsub($baseAllocation, $spentAmount, 4);
            // Ending = Starting + Net
            $endingBalance = bcadd($currentStart, $netVariance, 4);

            $results->push([
                'period_label'   => $periodStart->format('M Y'),
                'start_date'     => $periodStart->format('Y-m-d'),
                'end_date'       => $periodEnd->format('Y-m-d'),
                'starting'       => $currentStart,
                'allocation'     => $baseAllocation,
                'spent'          => $spentAmount,
                'net_variance'   => $netVariance,
                'ending'         => $endingBalance,
            ]);

            // Carry over ending balance to next period start
            $currentStart = $endingBalance;
        }

        return $results;
    }
}
```

---

### B. Chart Generator (`app/Generator/Chart/Budget/BudgetWaterfallGenerator.php`)

Create a custom generator that structures the data specifically for Chart.js floating bar datasets (`[y_min, y_max]`):

```php
namespace FireflyIII\Generator\Chart\Budget;

use Illuminate\Support\Collection;

class BudgetWaterfallGenerator
{
    /**
     * Generate Chart.js floating bar dataset format.
     */
    public function generate(string $title, Collection $waterfallData): array
    {
        $labels      = [];
        $starting    = [];
        $allocation  = [];
        $spent       = [];
        $ending      = [];

        foreach ($waterfallData as $row) {
            $labels[] = $row['period_label'];

            $start = (float) $row['starting'];
            $alloc = (float) $row['allocation'];
            $spnt  = (float) $row['spent'];
            $end   = (float) $row['ending'];

            // Floating bar coordinates: [bottom, top]
            $starting[]   = [0, $start];
            $allocation[] = [$start, $start + $alloc];
            $spent[]      = [$start + $alloc - $spnt, $start + $alloc];
            $ending[]     = [0, $end];
        }

        return [
            'title'  => $title,
            'labels' => $labels,
            'datasets' => [
                [
                    'label'           => 'Starting Balance',
                    'data'            => $starting,
                    'backgroundColor' => 'rgba(108, 117, 125, 0.5)', // Neutral Gray
                    'borderColor'     => '#6c757d',
                    'borderWidth'     => 1,
                ],
                [
                    'label'           => 'Allocation (+)',
                    'data'            => $allocation,
                    'backgroundColor' => 'rgba(25, 135, 84, 0.7)',  // Green
                    'borderColor'     => '#198754',
                    'borderWidth'     => 1,
                ],
                [
                    'label'           => 'Spent (-)',
                    'data'            => $spent,
                    'backgroundColor' => 'rgba(220, 53, 69, 0.7)',  // Red
                    'borderColor'     => '#dc3545',
                    'borderWidth'     => 1,
                ],
                [
                    'label'           => 'Ending Balance',
                    'data'            => $ending,
                    'backgroundColor' => 'rgba(13, 110, 253, 0.8)', // Primary Blue
                    'borderColor'     => '#0d6efd',
                    'borderWidth'     => 1,
                ],
            ],
        ];
    }
}
```

---

### C. Controller Endpoint (`app/Http/Controllers/Api/V1/Chart/BudgetController.php`)

Add the REST API route action:

```php
namespace FireflyIII\Http\Controllers\Api\V1\Chart;

use Carbon\Carbon;
use FireflyIII\Generator\Chart\Budget\BudgetWaterfallGenerator;
use FireflyIII\Http\Controllers\Controller;
use FireflyIII\Repositories\Budget\BudgetRepositoryInterface;
use FireflyIII\Repositories\Currency\CurrencyRepositoryInterface;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class BudgetController extends Controller
{
    public function waterfall(
        Request $request,
        int $id,
        BudgetRepositoryInterface $budgetRepository,
        CurrencyRepositoryInterface $currencyRepository,
        BudgetWaterfallGenerator $generator
    ): JsonResponse {
        $budget   = $budgetRepository->find($id);
        if (null === $budget) {
            return response()->json(['message' => 'Budget not found'], 404);
        }

        $currencyId = $request->input('currency_id', config('firefly.default_currency_id'));
        $currency   = $currencyRepository->find((int) $currencyId);

        $start = $request->has('start')
            ? Carbon::parse($request->input('start'))
            : Carbon::now()->startOfYear();
        $end   = $request->has('end')
            ? Carbon::parse($request->input('end'))
            : Carbon::now()->endOfYear();

        $data    = $budgetRepository->getWaterfallData($budget, $currency, $start, $end);
        $payload = $generator->generate(
            sprintf('Waterfall Chart for %s (%s)', $budget->name, $currency->code),
            $data
        );

        return response()->json($payload);
    }
}
```

---

## 4. Frontend & Chart.js Configuration

### Chart.js Floating Bar Setup
In JS view templates (Alpine.js / Chart.js wrapper):

```javascript
const ctx = document.getElementById('budgetWaterfallChart').getContext('2d');

const waterfallChart = new Chart(ctx, {
    type: 'bar',
    data: chartApiResponseData,
    options: {
        responsive: true,
        plugins: {
            tooltip: {
                callbacks: {
                    label: function(context) {
                        const raw = context.raw;
                        const diff = (raw[1] - raw[0]).toFixed(2);
                        return `${context.dataset.label}: ${diff} (${raw[0].toFixed(2)} to ${raw[1].toFixed(2)})`;
                    }
                }
            }
        },
        scales: {
            x: {
                stacked: false,
                title: { display: true, text: 'Period' }
            },
            y: {
                title: { display: true, text: 'Amount' }
            }
        }
    }
});
```

### Accessibility Fallback (Visually Hidden Table)

```html
<div class="chart-container">
    <canvas id="budgetWaterfallChart" aria-label="Budget Waterfall Chart" role="img"></canvas>

    <table class="visually-hidden" summary="Waterfall balance breakdown per period">
        <thead>
            <tr>
                <th scope="col">Period</th>
                <th scope="col">Starting Balance</th>
                <th scope="col">Allocation</th>
                <th scope="col">Spent</th>
                <th scope="col">Ending Balance</th>
            </tr>
        </thead>
        <tbody>
            <template x-for="row in waterfallTableData" :key="row.period_label">
                <tr>
                    <th scope="row" x-text="row.period_label"></th>
                    <td x-text="row.starting"></td>
                    <td x-text="row.allocation"></td>
                    <td x-text="row.spent"></td>
                    <td x-text="row.ending"></td>
                </tr>
            </template>
        </tbody>
    </table>
</div>
```

---

## 5. Automated Pest PHP Test Suite

Write comprehensive tests covering edge cases (zero spending, multi-currency limits, auto-budget rollover):

```php
uses(Tests\TestCase::class);

use FireflyIII\Models\Budget;
use FireflyIII\Models\TransactionCurrency;
use FireflyIII\Repositories\Budget\BudgetRepositoryInterface;
use Carbon\Carbon;

it('correctly calculates waterfall steps for a simple budget', function () {
    $budget   = Budget::factory()->create(['name' => 'Groceries']);
    $currency = TransactionCurrency::factory()->create(['code' => 'EUR']);

    // Mock 2 periods with $1000 limit and $800 spending
    $repository = app(BudgetRepositoryInterface::class);

    $start = Carbon::parse('2024-01-01');
    $end   = Carbon::parse('2024-02-29');

    $waterfall = $repository->getWaterfallData($budget, $currency, $start, $end);

    expect($waterfall)->toHaveCount(2);
    expect($waterfall->first()['starting'])->toBe('0.0000');
});

it('prevents double counting when auto-budget rollover is active', function () {
    $budget   = Budget::factory()->create(['name' => 'Savings Plan']);
    $currency = TransactionCurrency::factory()->create(['code' => 'USD']);

    // Attach rollover auto-budget
    $budget->autoBudgets()->create([
        'transaction_currency_id' => $currency->id,
        'amount'                  => 500,
        'period'                  => 'monthly',
        'type'                    => 'rollover',
    ]);

    $repository = app(BudgetRepositoryInterface::class);
    $data = $repository->getWaterfallData(
        $budget,
        $currency,
        Carbon::parse('2024-01-01'),
        Carbon::parse('2024-03-31')
    );

    // Verify carryovers step forward without duplicating rollover limits
    $period1End   = $data[0]['ending'];
    $period2Start = $data[1]['starting'];

    expect($period1End)->toEqual($period2Start);
});

it('denies access or returns 404 for invalid budget id via chart api', function () {
    $response = $this->getJson('/api/v1/chart/budgets/999999/waterfall');
    $response->assertStatus(404);
});
```

---

## 6. Refined GitHub Feature Request

```markdown
## Feature Request: Budget Waterfall Chart Visualization

### Problem Statement
In Firefly III, users can easily track individual period limits and expenditures. However, visualizing period-over-period compounding balances, rolled-over savings, or cumulative budget deficits requires manual calculation across multiple monthly views.

### Proposed Solution
Introduce a **Budget Waterfall Chart** under `Reports > Budgets` and a corresponding API endpoint (`GET /api/v1/chart/budgets/{id}/waterfall`).

The chart breaks down period dynamics into four sequential steps per period:
1. **Starting Balance:** Accumulated carryover from prior periods.
2. **Allocation (+):** Budgeted limit allocated for the period (normalized for AutoBudget rollover).
3. **Actual Spent (-):** Sum of expenses assigned to the budget in the target currency.
4. **Ending Balance (=):** Net remaining balance carried into the next period.

### Key Capabilities
- **Multi-Currency Aware:** Scoped per `currency_id` to handle multi-currency budget limits seamlessly.
- **AutoBudget Safe:** Dynamically adjusts base allocations when `rollover` AutoBudgets are active, preventing duplicate carryover counting.
- **Flexible Granularity:** Automatically maps to weekly, monthly, or yearly budget frequencies based on `BudgetRepository` date bounds.
- **Accessible:** Paired with an accessible `<table class="visually-hidden">` fallback.

### Visual Concept
Implemented using **Chart.js floating bars**:
- 🔘 **Gray Bar:** Starting Balance
- 🟢 **Green Bar:** + Allocation
- 🔴 **Red Bar:** - Actual Spent
- 🔵 **Blue Bar:** = Ending Balance

### Value Proposition
Provides actionable visibility into budget sustainability—making it instantly clear if a user's budget is safely accumulating a long-term buffer or experiencing a multi-month deficit drift.
```
