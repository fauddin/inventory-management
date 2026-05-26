<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Pick a budget and we'll recommend the highest-impact items to restock from your demand forecast.</p>
    </div>

    <div v-if="loading" class="loading">Loading demand forecasts...</div>
    <div v-else-if="error && !lastSubmitted" class="error">{{ error }}</div>
    <div v-else>
      <!-- Success banner after submission -->
      <div v-if="lastSubmitted" class="success-banner">
        <div class="success-banner-content">
          <div class="success-title">Order Submitted Successfully</div>
          <div class="success-details">
            <span><strong>Order:</strong> {{ lastSubmitted.order_number }}</span>
            <span><strong>Expected Delivery:</strong> {{ formatDate(lastSubmitted.expected_delivery) }}</span>
            <span><strong>Lead Time:</strong> {{ lastSubmitted.lead_time_days }} days</span>
          </div>
          <router-link to="/orders" class="view-orders-link">View in Orders</router-link>
        </div>
      </div>

      <!-- Error shown after a failed submission attempt (lastSubmitted may still be set from prev) -->
      <div v-if="error && lastSubmitted" class="error">{{ error }}</div>

      <!-- Stats row -->
      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">Budget</div>
          <div class="stat-value">${{ formatCurrency(budget) }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">Items Recommended</div>
          <div class="stat-value">{{ recommendedItems.length }}</div>
        </div>
        <div class="stat-card" :class="totalCost > 0 ? 'warning' : ''">
          <div class="stat-label">Total Cost</div>
          <div class="stat-value">${{ formatCurrency(totalCost) }}</div>
        </div>
        <div class="stat-card" :class="budgetRemaining < 0 ? 'danger' : 'success'">
          <div class="stat-label">Budget Remaining</div>
          <div class="stat-value">${{ formatCurrency(budgetRemaining) }}</div>
        </div>
      </div>

      <!-- Budget slider card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Set Your Budget</h3>
        </div>
        <div class="slider-container">
          <input
            type="range"
            min="0"
            max="200000"
            step="1000"
            v-model.number="budget"
            class="budget-slider"
          />
          <div class="slider-labels">
            <span>$0</span>
            <span class="slider-current">${{ formatCurrency(budget) }}</span>
            <span>$200,000</span>
          </div>
        </div>
      </div>

      <!-- Recommended items card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
          <span v-if="recommendedItems.length > 0" class="items-count-badge">
            {{ recommendedItems.length }} item{{ recommendedItems.length !== 1 ? 's' : '' }}
          </span>
        </div>

        <div v-if="recommendedItems.length === 0" class="empty-state">
          Increase your budget to see recommendations.
        </div>

        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Name</th>
                <th>Trend</th>
                <th>Shortfall</th>
                <th>Unit Cost</th>
                <th>Quantity</th>
                <th>Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.item_sku">
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td>{{ item.shortfall }}</td>
                <td>${{ formatCurrency(item.unitCost) }}</td>
                <td>{{ item.shortfall }}</td>
                <td><strong>${{ formatCurrency(item.lineTotal) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Card footer with total and place order button -->
        <div class="card-footer">
          <div class="total-cost">
            Total: <strong>${{ formatCurrency(totalCost) }}</strong>
          </div>
          <button
            class="btn-primary"
            :disabled="recommendedItems.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Submitting...' : 'Place Order' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

// Demand forecast data lacks unit_cost and category. We derive these client-side
// from the SKU prefix so the recommendation algorithm has cost and lead-time info
// without requiring an additional inventory API call.
const SKU_PREFIX_META = {
  WDG: { unitCost:  18.50, category: 'Other' },          // Widget
  BRG: { unitCost:  42.00, category: 'Actuators' },      // Bearing
  GSK: { unitCost:   6.75, category: 'Other' },          // Gasket
  MTR: { unitCost: 285.00, category: 'Actuators' },      // Motor
  FLT: { unitCost:  14.25, category: 'Other' },          // Filter
  VLV: { unitCost:  98.50, category: 'Actuators' },      // Valve
  PSU: { unitCost: 175.00, category: 'Power Supplies' }, // Power supply
  SNR: { unitCost:  62.00, category: 'Sensors' },        // Sensor
  CTL: { unitCost: 230.00, category: 'Controllers' },    // Controller
}
// Fallback when a SKU prefix isn't recognized.
const FALLBACK_META = { unitCost: 40.00, category: 'Other' }

function lookupMeta(sku) {
  const prefix = (sku || '').slice(0, 3).toUpperCase()
  return SKU_PREFIX_META[prefix] || FALLBACK_META
}

export default {
  name: 'Restocking',
  setup() {
    const forecasts = ref([])
    const loading = ref(true)
    const error = ref(null)
    const budget = ref(50000)
    const submitting = ref(false)
    const lastSubmitted = ref(null)

    const loadForecasts = async () => {
      loading.value = true
      error.value = null
      try {
        forecasts.value = await api.getDemandForecasts()
      } catch (err) {
        error.value = 'Failed to load demand forecasts: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // Build candidate list, attach cost/category, filter to items with positive shortfall,
    // sort by line total descending (greedy: biggest-impact items first), then walk the
    // list and include each item that fits within the remaining budget.
    const recommendedItems = computed(() => {
      const candidates = forecasts.value
        .map(f => {
          const shortfall = Math.max(0, f.forecasted_demand - f.current_demand)
          if (shortfall === 0) return null
          const { unitCost, category } = lookupMeta(f.item_sku)
          return {
            ...f,
            shortfall,
            unitCost,
            category,
            lineTotal: shortfall * unitCost,
          }
        })
        .filter(Boolean)

      // Sort descending by line total so highest-impact items are considered first
      candidates.sort((a, b) => b.lineTotal - a.lineTotal)

      let remaining = budget.value
      const selected = []
      for (const item of candidates) {
        if (item.lineTotal <= remaining) {
          selected.push(item)
          remaining -= item.lineTotal
        }
      }
      return selected
    })

    const totalCost = computed(() =>
      recommendedItems.value.reduce((sum, item) => sum + item.lineTotal, 0)
    )

    const budgetRemaining = computed(() => budget.value - totalCost.value)

    const formatCurrency = (value) =>
      value.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })

    const formatDate = (dateString) => {
      if (!dateString) return '-'
      const d = new Date(dateString)
      if (isNaN(d.getTime())) return dateString
      return d.toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' })
    }

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0 || submitting.value) return
      submitting.value = true
      error.value = null
      try {
        const payload = {
          budget: budget.value,
          items: recommendedItems.value.map(item => ({
            sku: item.item_sku,
            name: item.item_name,
            category: item.category,
            quantity: item.shortfall,
            unit_cost: item.unitCost,
          })),
        }
        const order = await api.submitRestockOrder(payload)
        lastSubmitted.value = order
      } catch (err) {
        error.value = 'Failed to submit restock order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadForecasts)

    return {
      forecasts,
      loading,
      error,
      budget,
      submitting,
      lastSubmitted,
      recommendedItems,
      totalCost,
      budgetRemaining,
      formatCurrency,
      formatDate,
      placeOrder,
    }
  }
}
</script>

<style scoped>
/* Budget slider */
.slider-container {
  padding: 0.5rem 0 1rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  transition: box-shadow 0.15s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  box-shadow: 0 1px 8px rgba(37, 99, 235, 0.6);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: none;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 0.75rem;
  font-size: 0.813rem;
  color: #64748b;
}

.slider-current {
  font-size: 1rem;
  font-weight: 700;
  color: #2563eb;
}

/* Empty state */
.empty-state {
  text-align: center;
  padding: 2.5rem 1rem;
  color: #64748b;
  font-size: 0.938rem;
}

/* Card footer with total + button */
.card-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 1rem;
  margin-top: 0.5rem;
  border-top: 1px solid #e2e8f0;
}

.total-cost {
  font-size: 1rem;
  color: #334155;
}

.total-cost strong {
  font-size: 1.125rem;
  color: #0f172a;
}

/* Primary button */
.btn-primary {
  padding: 0.625rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease, opacity 0.2s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Items count badge in card header */
.items-count-badge {
  font-size: 0.813rem;
  font-weight: 600;
  color: #2563eb;
  background: #eff6ff;
  padding: 0.25rem 0.75rem;
  border-radius: 999px;
}

/* Success banner */
.success-banner {
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  border-radius: 10px;
  padding: 1.25rem 1.5rem;
  margin-bottom: 1.25rem;
}

.success-banner-content {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  flex-wrap: wrap;
}

.success-title {
  font-size: 1rem;
  font-weight: 700;
  color: #065f46;
  flex-shrink: 0;
}

.success-details {
  display: flex;
  gap: 1.5rem;
  flex-wrap: wrap;
  font-size: 0.875rem;
  color: #047857;
  flex: 1;
}

.view-orders-link {
  font-size: 0.875rem;
  font-weight: 600;
  color: #2563eb;
  text-decoration: none;
  white-space: nowrap;
  padding: 0.375rem 0.875rem;
  border: 1px solid #93c5fd;
  border-radius: 6px;
  background: white;
  transition: background 0.15s ease;
}

.view-orders-link:hover {
  background: #eff6ff;
}
</style>
