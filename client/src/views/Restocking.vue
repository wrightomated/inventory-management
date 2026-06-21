<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Automatically allocate your available budget to the highest-demand items.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Budget Slider Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>
        <div class="budget-section">
          <div class="budget-display">
            {{ formatCurrency(budget) }}
          </div>
          <input
            type="range"
            class="budget-slider"
            v-model.number="budget"
            min="0"
            max="50000"
            step="500"
          />
          <div class="budget-range-labels">
            <span>$0</span>
            <span>$50,000</span>
          </div>
        </div>
      </div>

      <!-- Recommendations Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Restock Recommendations ({{ positiveGapForecasts.length }} items)</h3>
          <span class="card-subtitle">Sorted by demand gap — items auto-selected to fit budget</span>
        </div>
        <div v-if="positiveGapForecasts.length === 0" class="empty-state">
          No items with positive demand gaps found.
        </div>
        <div v-else class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th class="col-name">Item Name</th>
                <th class="col-sku">SKU</th>
                <th class="col-gap">Demand Gap</th>
                <th class="col-cost">Unit Cost</th>
                <th class="col-qty">Restock Qty</th>
                <th class="col-total">Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="forecast in positiveGapForecasts"
                :key="forecast.id"
                :class="{ 'row-selected': selectedItems[forecast.id], 'row-over-budget': !selectedItems[forecast.id] && isOverBudgetIfAdded(forecast) }"
              >
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="selectedItems[forecast.id]"
                    @change="toggleItem(forecast)"
                  />
                </td>
                <td class="col-name"><strong>{{ forecast.item_name }}</strong></td>
                <td class="col-sku"><code>{{ forecast.item_sku }}</code></td>
                <td class="col-gap">
                  <span class="gap-badge">+{{ getGap(forecast) }}</span>
                </td>
                <td class="col-cost">{{ formatCurrency(forecast.unit_cost) }}</td>
                <td class="col-qty">{{ getGap(forecast) }}</td>
                <td class="col-total"><strong>{{ formatCurrency(getLineTotal(forecast)) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Summary / Order Card -->
      <div class="card summary-card">
        <div class="card-header">
          <h3 class="card-title">Order Summary</h3>
        </div>
        <div class="summary-grid">
          <div class="summary-stat">
            <div class="summary-label">Selected Items</div>
            <div class="summary-value">{{ selectedForecasts.length }}</div>
          </div>
          <div class="summary-stat">
            <div class="summary-label">Total Cost</div>
            <div class="summary-value">{{ formatCurrency(totalCost) }}</div>
          </div>
          <div class="summary-stat" :class="{ 'over-budget': remainingBudget < 0 }">
            <div class="summary-label">Remaining Budget</div>
            <div class="summary-value">{{ formatCurrency(remainingBudget) }}</div>
          </div>
        </div>

        <div class="order-actions">
          <button
            class="btn-place-order"
            :disabled="selectedForecasts.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Submitting...' : 'Place Order' }}
          </button>
          <div v-if="successMessage" class="success-message">{{ successMessage }}</div>
          <div v-if="submitError" class="error">{{ submitError }}</div>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const allForecasts = ref([])
    const budget = ref(10000)
    // Keys are forecast IDs; value true means checked
    const selectedItems = ref({})
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const successMessage = ref('')
    const submitError = ref('')

    // Only forecasts where demand gap is positive, sorted descending by gap
    const positiveGapForecasts = computed(() => {
      return allForecasts.value
        .filter(f => f.forecasted_demand > f.current_demand)
        .sort((a, b) => getGap(b) - getGap(a))
    })

    const getGap = (forecast) => forecast.forecasted_demand - forecast.current_demand

    const getLineTotal = (forecast) => getGap(forecast) * forecast.unit_cost

    const selectedForecasts = computed(() =>
      positiveGapForecasts.value.filter(f => selectedItems.value[f.id])
    )

    const totalCost = computed(() =>
      selectedForecasts.value.reduce((sum, f) => sum + getLineTotal(f), 0)
    )

    const remainingBudget = computed(() => budget.value - totalCost.value)

    // Greedy auto-select: check items from top of list until budget is exceeded
    const autoSelectByBudget = () => {
      const newSelections = {}
      let running = 0
      for (const forecast of positiveGapForecasts.value) {
        const cost = getLineTotal(forecast)
        if (running + cost <= budget.value) {
          newSelections[forecast.id] = true
          running += cost
        }
        // Items that don't fit are simply left unchecked
      }
      selectedItems.value = newSelections
    }

    // Whether adding this item would push total over budget (for row styling)
    const isOverBudgetIfAdded = (forecast) => {
      return totalCost.value + getLineTotal(forecast) > budget.value
    }

    // Manual toggle — user overrides auto-selection
    const toggleItem = (forecast) => {
      const current = { ...selectedItems.value }
      if (current[forecast.id]) {
        delete current[forecast.id]
      } else {
        current[forecast.id] = true
      }
      selectedItems.value = current
    }

    // Re-run auto-select whenever budget changes
    watch(budget, autoSelectByBudget)

    const formatCurrency = (value) => {
      if (value == null || isNaN(value)) return '$0.00'
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD' })
    }

    const loadForecasts = async () => {
      loading.value = true
      error.value = null
      try {
        allForecasts.value = await api.getDemandForecasts()
        // Run initial auto-selection after data loads
        autoSelectByBudget()
      } catch (err) {
        error.value = 'Failed to load demand forecasts: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      submitError.value = ''
      successMessage.value = ''
      try {
        const items = selectedForecasts.value.map(f => ({
          sku: f.item_sku,
          name: f.item_name,
          quantity: getGap(f),
          unit_cost: f.unit_cost,
          line_total: getLineTotal(f)
        }))
        await api.createRestockingOrder({
          items,
          total_cost: totalCost.value,
          budget: budget.value
        })
        successMessage.value = 'Order submitted successfully!'
        // Reset to defaults
        budget.value = 10000
        selectedItems.value = {}
      } catch (err) {
        submitError.value = 'Failed to submit order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadForecasts)

    return {
      allForecasts,
      budget,
      selectedItems,
      loading,
      error,
      submitting,
      successMessage,
      submitError,
      positiveGapForecasts,
      selectedForecasts,
      totalCost,
      remainingBudget,
      getGap,
      getLineTotal,
      isOverBudgetIfAdded,
      toggleItem,
      formatCurrency,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  /* page uses global .main-content padding */
}

/* Budget slider */
.budget-section {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  padding: 0.5rem 0;
}

.budget-display {
  font-size: 2.25rem;
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  max-width: 600px;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  max-width: 600px;
  font-size: 0.75rem;
  color: var(--text-muted);
}

.card-subtitle {
  font-size: 0.813rem;
  color: var(--text-muted);
  font-weight: 400;
}

/* Table */
.restock-table {
  table-layout: fixed;
  width: 100%;
  border-collapse: collapse;
}

.col-check { width: 44px; }
.col-name  { width: 220px; }
.col-sku   { width: 130px; }
.col-gap   { width: 120px; }
.col-cost  { width: 110px; }
.col-qty   { width: 110px; }
.col-total { width: 130px; }

.restock-table thead {
  background: var(--bg-surface-raised);
  border-top: 1px solid var(--border);
  border-bottom: 1px solid var(--border-strong);
}

.restock-table th {
  text-align: left;
  padding: 0.5rem 0.75rem;
  font-weight: 600;
  color: var(--text-muted);
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.restock-table td {
  padding: 0.5rem 0.75rem;
  border-top: 1px solid var(--border);
  color: var(--text-primary);
  font-size: 0.875rem;
}

.restock-table tbody tr {
  transition: background-color 0.15s ease;
}

.restock-table tbody tr:hover {
  background: var(--bg-surface-hover);
}

.row-selected {
  background: var(--info-dim);
}

.row-selected:hover {
  background: rgba(56, 139, 253, 0.25) !important;
}

/* Dim rows that would push over budget */
.row-over-budget {
  opacity: 0.5;
}

.gap-badge {
  display: inline-block;
  padding: 0.188rem 0.5rem;
  background: var(--warning-dim);
  color: var(--warning);
  border-radius: 4px;
  font-size: 0.813rem;
  font-weight: 600;
}

code {
  font-family: 'JetBrains Mono', 'Fira Code', monospace;
  font-size: 0.813rem;
  color: var(--text-muted);
}

/* Empty state */
.empty-state {
  padding: 2rem;
  text-align: center;
  color: var(--text-muted);
  font-size: 0.938rem;
}

/* Summary */
.summary-card {
  border-top: 3px solid #2563eb;
}

.summary-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 1.25rem;
  margin-bottom: 1.5rem;
}

.summary-stat {
  padding: 1rem;
  background: var(--bg-surface-raised);
  border-radius: 8px;
  border: 1px solid var(--border);
}

.summary-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: var(--text-muted);
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.5rem;
}

.summary-value {
  font-size: 1.75rem;
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: -0.025em;
}

.over-budget .summary-value {
  color: #dc2626;
}

/* Actions */
.order-actions {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.btn-place-order {
  padding: 0.75rem 2rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.btn-place-order:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-place-order:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-message {
  padding: 0.625rem 1rem;
  background: var(--success-dim);
  color: var(--success);
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 500;
}
</style>
