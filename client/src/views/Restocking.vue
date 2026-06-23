<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set your available budget and place a restocking order based on demand forecasts.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>
        <div class="budget-display">{{ formatCurrency(budget) }}</div>
        <input
          type="range"
          min="0"
          max="50000"
          step="500"
          v-model.number="budget"
          class="budget-slider"
        />
        <div class="budget-labels">
          <span>$0</span>
          <span>$50,000</span>
        </div>
      </div>

      <!-- Recommended Items Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ recommendedItems.length }})</h3>
        </div>

        <div v-if="budget === 0" class="empty-state">
          Set a budget above to see recommendations.
        </div>
        <div v-else-if="recommendedItems.length === 0" class="empty-state">
          No items fit within the current budget.
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Trend</th>
                <th>Qty to Restock</th>
                <th>Unit Cost</th>
                <th>Est. Cost</th>
                <th class="col-include">Include</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.item_sku">
                <td>
                  {{ item.item_name }}
                  <span v-if="item.unit_cost === 0" class="cost-tbd">Cost TBD</span>
                </td>
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td>{{ item.gap.toLocaleString() }}</td>
                <td>{{ item.unit_cost === 0 ? '—' : formatCurrency(item.unit_cost) }}</td>
                <td>{{ item.unit_cost === 0 ? '—' : formatCurrency(item.estimated_cost) }}</td>
                <td class="col-include">
                  <input
                    type="checkbox"
                    :checked="isSelected(item.item_sku)"
                    @change="toggleItem(item.item_sku)"
                    class="item-checkbox"
                  />
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Order Summary / Success Card -->
      <div v-if="submittedOrder" class="card success-card">
        <div class="card-header">
          <h3 class="card-title success-title">Order Submitted</h3>
        </div>
        <div class="success-body">
          <div class="success-row">
            <span class="success-label">Order Number</span>
            <span class="success-value">{{ submittedOrder.order_number }}</span>
          </div>
          <div class="success-row">
            <span class="success-label">Expected Delivery</span>
            <span class="success-value">{{ formatDate(submittedOrder.expected_delivery) }}</span>
          </div>
          <div class="success-actions">
            <router-link to="/orders" class="link-orders">View in Orders tab</router-link>
            <button class="btn-secondary" @click="resetState">Place Another Order</button>
          </div>
        </div>
      </div>

      <div v-else-if="recommendedItems.length > 0" class="card">
        <div class="card-header">
          <h3 class="card-title">Order Summary</h3>
        </div>
        <div class="summary-body">
          <div class="summary-row">
            <span class="summary-label">Selected Items</span>
            <span class="summary-value">{{ selectedItemsForOrder.length }}</span>
          </div>
          <div class="summary-row">
            <span class="summary-label">Total Estimated Cost</span>
            <span class="summary-value">{{ formatCurrency(totalSelectedCost) }}</span>
          </div>
          <div class="summary-row">
            <span class="summary-label">Remaining Budget After Order</span>
            <span class="summary-value" :class="{ 'value-negative': budget - totalSelectedCost < 0 }">
              {{ formatCurrency(budget - totalSelectedCost) }}
            </span>
          </div>
        </div>
        <div class="summary-actions">
          <button
            class="btn-primary"
            :disabled="selectedItemsForOrder.length === 0 || budget === 0 || submitting"
            @click="submitOrder"
          >
            {{ submitting ? 'Submitting...' : 'Place Order' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const router = useRouter()

    const budget = ref(10000)
    const allForecasts = ref([])
    const inventoryItems = ref([])
    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const submittedOrder = ref(null)
    const selectedSkus = ref({})

    // Build SKU -> unit_cost map from inventory
    const skuCostMap = computed(() => {
      const map = {}
      inventoryItems.value.forEach(item => {
        map[item.sku] = item.unit_cost
      })
      return map
    })

    // Trend priority order for sorting
    const trendPriority = { increasing: 0, stable: 1, decreasing: 2 }

    // Greedy recommendation algorithm
    const recommendedItems = computed(() => {
      if (budget.value === 0) return []

      // Sort by trend priority
      const sorted = [...allForecasts.value].sort((a, b) => {
        return (trendPriority[a.trend] ?? 3) - (trendPriority[b.trend] ?? 3)
      })

      const result = []
      let runningCost = 0

      for (const forecast of sorted) {
        const gap = forecast.forecasted_demand - forecast.current_demand
        if (gap <= 0) continue

        const unit_cost = skuCostMap.value[forecast.item_sku] ?? 0
        const estimated_cost = gap * unit_cost

        // Items with unit_cost = 0 don't count against budget
        if (unit_cost > 0 && runningCost + estimated_cost > budget.value) {
          continue
        }

        runningCost += estimated_cost
        result.push({
          ...forecast,
          gap,
          unit_cost,
          estimated_cost
        })
      }

      return result
    })

    // Reset selectedSkus whenever recommendedItems changes
    watch(recommendedItems, (items) => {
      const map = {}
      items.forEach(item => {
        map[item.item_sku] = true
      })
      selectedSkus.value = map
    })

    const isSelected = (sku) => {
      return !!selectedSkus.value[sku]
    }

    const toggleItem = (sku) => {
      selectedSkus.value = {
        ...selectedSkus.value,
        [sku]: !selectedSkus.value[sku]
      }
    }

    // Only forecasts that are selected
    const selectedItemsForOrder = computed(() => {
      return recommendedItems.value.filter(item => isSelected(item.item_sku))
    })

    const totalSelectedCost = computed(() => {
      return selectedItemsForOrder.value.reduce((sum, item) => sum + item.estimated_cost, 0)
    })

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])
        allForecasts.value = forecastsData
        inventoryItems.value = inventoryData
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const submitOrder = async () => {
      if (selectedItemsForOrder.value.length === 0 || budget.value === 0 || submitting.value) return

      submitting.value = true
      error.value = null
      try {
        const itemsPayload = selectedItemsForOrder.value.map(f => ({
          sku: f.item_sku,
          name: f.item_name,
          quantity: f.gap,
          unit_price: skuCostMap.value[f.item_sku] ?? 0
        }))
        const result = await api.submitRestockingOrder(itemsPayload)
        submittedOrder.value = result
      } catch (err) {
        error.value = 'Failed to submit order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    const resetState = () => {
      submittedOrder.value = null
      budget.value = 10000
    }

    const formatCurrency = (value) => {
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 })
    }

    const formatDate = (dateStr) => {
      if (!dateStr) return '—'
      const d = new Date(dateStr)
      if (isNaN(d.getTime())) return dateStr
      return d.toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' })
    }

    onMounted(loadData)

    return {
      budget,
      allForecasts,
      inventoryItems,
      loading,
      error,
      submitting,
      submittedOrder,
      selectedSkus,
      recommendedItems,
      selectedItemsForOrder,
      totalSelectedCost,
      isSelected,
      toggleItem,
      submitOrder,
      resetState,
      formatCurrency,
      formatDate
    }
  }
}
</script>

<style scoped>
.budget-display {
  font-size: 2.5rem;
  font-weight: 700;
  color: #2563eb;
  margin-bottom: 1rem;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  height: 6px;
  cursor: pointer;
  display: block;
}

.budget-labels {
  display: flex;
  justify-content: space-between;
  margin-top: 0.5rem;
  color: #64748b;
  font-size: 0.813rem;
}

.empty-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.col-include {
  text-align: center;
}

.item-checkbox {
  width: 16px;
  height: 16px;
  cursor: pointer;
  accent-color: #2563eb;
}

.cost-tbd {
  display: inline-block;
  margin-left: 0.5rem;
  font-size: 0.75rem;
  color: #94a3b8;
  font-style: italic;
}

.summary-body {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  margin-bottom: 1.25rem;
}

.summary-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.5rem 0;
  border-bottom: 1px solid #f1f5f9;
}

.summary-row:last-child {
  border-bottom: none;
}

.summary-label {
  color: #64748b;
  font-size: 0.875rem;
  font-weight: 500;
}

.summary-value {
  font-size: 0.938rem;
  font-weight: 600;
  color: #0f172a;
}

.value-negative {
  color: #dc2626;
}

.summary-actions {
  padding-top: 0.75rem;
  border-top: 1px solid #e2e8f0;
}

.btn-primary {
  background: #2563eb;
  color: white;
  padding: 0.625rem 1.5rem;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.938rem;
  cursor: pointer;
  transition: background 0.2s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #93c5fd;
  cursor: not-allowed;
}

.success-card {
  background: #f0fdf4;
  border-color: #bbf7d0;
}

.success-title {
  color: #065f46;
}

.success-body {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.success-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.5rem 0;
  border-bottom: 1px solid #d1fae5;
}

.success-row:last-child {
  border-bottom: none;
}

.success-label {
  color: #064e3b;
  font-size: 0.875rem;
  font-weight: 500;
}

.success-value {
  font-size: 0.938rem;
  font-weight: 600;
  color: #065f46;
}

.success-actions {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding-top: 1rem;
  border-top: 1px solid #bbf7d0;
  flex-wrap: wrap;
}

.link-orders {
  color: #2563eb;
  font-weight: 600;
  text-decoration: none;
  font-size: 0.938rem;
}

.link-orders:hover {
  text-decoration: underline;
}

.btn-secondary {
  background: white;
  color: #374151;
  padding: 0.625rem 1.5rem;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.938rem;
  cursor: pointer;
  transition: all 0.2s ease;
}

.btn-secondary:hover {
  background: #f9fafb;
  border-color: #9ca3af;
}
</style>
