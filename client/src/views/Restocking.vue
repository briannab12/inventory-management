<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Recommend and order inventory restocking based on demand forecasts and stock levels</p>
    </div>

    <div v-if="successMessage" class="success-toast">{{ successMessage }}</div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="stats-grid">
        <div class="stat-card danger">
          <div class="stat-label">Low Stock Items</div>
          <div class="stat-value">{{ allRecommendations.filter(r => r.priority === 'high').length }}</div>
        </div>
        <div class="stat-card info">
          <div class="stat-label">Items Within Budget</div>
          <div class="stat-value">{{ selectedRecommendations.length }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">Total Order Cost</div>
          <div class="stat-value">{{ currencySymbol }}{{ totalOrderCost.toLocaleString(undefined, { maximumFractionDigits: 0 }) }}</div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>
        <div class="budget-section">
          <div class="budget-display">
            <span class="budget-label">Budget:</span>
            <span class="budget-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
          </div>
          <div class="slider-container">
            <input
              type="range"
              class="budget-slider"
              :min="sliderMin"
              :max="sliderMax"
              :value="budget"
              step="100"
              @input="budget = Number($event.target.value)"
            />
            <div class="slider-labels">
              <span>{{ currencySymbol }}{{ sliderMin.toLocaleString() }}</span>
              <span>{{ currencySymbol }}{{ sliderMax.toLocaleString() }}</span>
            </div>
          </div>
          <p class="budget-hint">
            {{ selectedRecommendations.length }} item{{ selectedRecommendations.length !== 1 ? 's' : '' }} selected
            totaling {{ currencySymbol }}{{ totalOrderCost.toLocaleString(undefined, { maximumFractionDigits: 0 }) }}
            within your {{ currencySymbol }}{{ budget.toLocaleString() }} budget
          </p>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommendations ({{ allRecommendations.length }})</h3>
        </div>
        <div class="table-container">
          <table class="recommendations-table">
            <thead>
              <tr>
                <th>Priority</th>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Qty On Hand</th>
                <th>Reorder Point</th>
                <th>Unit Cost</th>
                <th>Suggested Qty</th>
                <th>Est. Cost</th>
                <th>Trend</th>
                <th>Lead Time</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="rec in allRecommendations"
                :key="rec.sku"
                :class="{ 'row-excluded': !selectedSkus.has(rec.sku) }"
              >
                <td><span :class="['badge', rec.priority]">{{ rec.priority.charAt(0).toUpperCase() + rec.priority.slice(1) }}</span></td>
                <td><code>{{ rec.sku }}</code></td>
                <td>{{ rec.name }}</td>
                <td :class="{ 'low-stock': rec.qty_on_hand <= rec.reorder_point }">{{ rec.qty_on_hand }}</td>
                <td>{{ rec.reorder_point }}</td>
                <td>{{ currencySymbol }}{{ rec.unit_cost.toLocaleString() }}</td>
                <td><strong>{{ rec.suggested_quantity }}</strong></td>
                <td :class="['est-cost', selectedSkus.has(rec.sku) ? 'within-budget' : 'over-budget']">
                  <strong>{{ currencySymbol }}{{ rec.estimated_cost.toLocaleString(undefined, { maximumFractionDigits: 0 }) }}</strong>
                </td>
                <td>
                  <span v-if="rec.trend" :class="['badge', rec.trend]">{{ rec.trend }}</span>
                  <span v-else class="no-data">—</span>
                </td>
                <td>{{ rec.lead_time_days }} days</td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="action-bar">
          <div class="action-summary">
            {{ selectedRecommendations.length }} items selected &bull;
            Total: <strong>{{ currencySymbol }}{{ totalOrderCost.toLocaleString(undefined, { maximumFractionDigits: 0 }) }}</strong>
          </div>
          <button
            class="place-order-btn"
            :disabled="selectedRecommendations.length === 0 || submitting"
            @click="placeOrder"
          >
            <span v-if="submitting">Placing Order...</span>
            <span v-else>Place Order</span>
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, watch, computed } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()
    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const successMessage = ref('')
    const allRecommendations = ref([])
    const budget = ref(0)

    const sliderMax = computed(() => {
      const total = allRecommendations.value.reduce((s, r) => s + r.estimated_cost, 0)
      return Math.ceil(total / 1000) * 1000 || 10000
    })

    const sliderMin = computed(() => 500)

    const selectedRecommendations = computed(() => {
      const sorted = [...allRecommendations.value]
        .sort((a, b) => b.priority_score - a.priority_score || a.estimated_cost - b.estimated_cost)
      let remaining = budget.value
      const selected = []
      for (const rec of sorted) {
        if (rec.estimated_cost <= remaining) {
          selected.push(rec)
          remaining -= rec.estimated_cost
        }
      }
      return selected
    })

    const selectedSkus = computed(() => new Set(selectedRecommendations.value.map(r => r.sku)))
    const totalOrderCost = computed(() => selectedRecommendations.value.reduce((s, r) => s + r.estimated_cost, 0))
    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        const filters = getCurrentFilters()
        allRecommendations.value = await api.getRestockingRecommendations(filters)
        if (budget.value === 0) {
          budget.value = Math.ceil(sliderMax.value * 0.25 / 100) * 100
        }
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (selectedRecommendations.value.length === 0 || submitting.value) return
      try {
        submitting.value = true
        error.value = null
        await api.createRestockingOrder({
          items: selectedRecommendations.value.map(r => ({
            sku: r.sku,
            name: r.name,
            quantity: r.suggested_quantity,
            unit_cost: r.unit_cost
          })),
          warehouse: selectedLocation.value !== 'all' ? selectedLocation.value : null
        })
        successMessage.value = 'Restocking order placed successfully!'
        setTimeout(() => { successMessage.value = '' }, 3000)
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    watch([selectedLocation, selectedCategory], loadRecommendations)

    onMounted(loadRecommendations)

    return {
      loading,
      error,
      submitting,
      successMessage,
      allRecommendations,
      budget,
      sliderMax,
      sliderMin,
      selectedRecommendations,
      selectedSkus,
      totalOrderCost,
      currencySymbol,
      placeOrder
    }
  }
}
</script>

<style scoped>
.success-toast {
  background: #dcfce7;
  color: #166534;
  border: 1px solid #bbf7d0;
  border-radius: 8px;
  padding: 0.75rem 1rem;
  margin-bottom: 1.5rem;
  font-weight: 500;
}

.budget-section {
  padding: 1.25rem 1.5rem 1.5rem;
}

.budget-display {
  display: flex;
  align-items: baseline;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.budget-label {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
}

.budget-value {
  font-size: 1.75rem;
  font-weight: 700;
  color: #0f172a;
}

.slider-container {
  margin-bottom: 0.75rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #3b82f6;
  cursor: pointer;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
  margin-top: 0.375rem;
}

.budget-hint {
  font-size: 0.875rem;
  color: #64748b;
  margin: 0;
}

.recommendations-table {
  table-layout: fixed;
  width: 100%;
}

.row-excluded {
  opacity: 0.4;
  background: #f8fafc;
}

.low-stock {
  color: #dc2626;
  font-weight: 600;
}

.est-cost.within-budget {
  color: #059669;
}

.est-cost.over-budget {
  color: #64748b;
}

.no-data {
  color: #94a3b8;
}

code {
  font-family: 'SF Mono', 'Fira Code', monospace;
  font-size: 0.8rem;
  background: #f1f5f9;
  padding: 0.15rem 0.4rem;
  border-radius: 3px;
  color: #0f172a;
}

.action-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem 1.5rem;
  border-top: 1px solid #e2e8f0;
  background: #f8fafc;
  border-radius: 0 0 8px 8px;
}

.action-summary {
  font-size: 0.875rem;
  color: #64748b;
}

.action-summary strong {
  color: #0f172a;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 0.625rem 1.5rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}
</style>
