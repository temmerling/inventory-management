<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div class="card">
      <label class="budget-label">{{ t('restocking.budgetLabel') }}</label>
      <div class="budget-display">{{ formatCurrency(budget) }}</div>
      <input
        type="range"
        class="budget-slider"
        v-model.number="budget"
        min="0"
        max="500000"
        step="1000"
      />
      <div class="budget-remaining">
        {{ t('restocking.budgetRemaining') }}: <strong>{{ formatCurrency(budgetRemaining) }}</strong>
      </div>
    </div>

    <div class="card">
      <div class="card-header">
        <h3 class="card-title">{{ t('restocking.recommendations') }}</h3>
      </div>

      <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <template v-else>
        <p v-if="budget === 0 || recommendations.length === 0" class="no-items">
          {{ recommendations.length === 0 ? t('restocking.noMatches') : t('restocking.noItems') }}
        </p>
        <template v-else>
          <div class="table-container">
            <table>
              <thead>
                <tr>
                  <th>SKU</th>
                  <th>{{ t('demand.table.itemName') }}</th>
                  <th>{{ t('restocking.trend') }}</th>
                  <th>{{ t('restocking.quantity') }}</th>
                  <th>{{ t('restocking.unitCost') }}</th>
                  <th>{{ t('restocking.totalCost') }}</th>
                  <th>Status</th>
                </tr>
              </thead>
              <tbody>
                <tr
                  v-for="item in recommendations"
                  :key="item.item_sku"
                  :class="{ 'row-dimmed': !item.inBudget }"
                >
                  <td><strong>{{ item.item_sku }}</strong></td>
                  <td>{{ item.item_name }}</td>
                  <td>
                    <span class="badge" :class="item.trend">{{ item.trend }}</span>
                  </td>
                  <td>{{ item.forecasted_demand.toLocaleString() }}</td>
                  <td>{{ formatCurrency(item.unit_cost) }}</td>
                  <td>{{ formatCurrency(item.total_cost) }}</td>
                  <td>
                    <span v-if="item.inBudget" class="badge success">{{ t('restocking.inBudget') }}</span>
                    <span v-else class="badge over-budget">{{ t('restocking.overBudget') }}</span>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>

          <div class="order-actions">
            <template v-if="!submittedOrder">
              <button
                class="place-order-btn"
                :disabled="inBudgetItems.length === 0 || submitting || !!submittedOrder"
                @click="placeOrder"
              >
                {{ submitting ? 'Placing order...' : t('restocking.placeOrder') }}
              </button>
            </template>

            <div v-if="submittedOrder" class="success-box">
              <div class="success-title">{{ t('restocking.orderPlaced') }}</div>
              <div class="success-detail">
                <span>{{ t('restocking.orderNumber') }}: <strong>{{ submittedOrder.order_number }}</strong></span>
                <span>{{ t('restocking.expectedDelivery') }}: <strong>{{ formatDate(submittedOrder.expected_delivery) }}</strong></span>
                <span>{{ t('restocking.estimatedCost') }}: <strong>{{ formatCurrency(submittedOrder.total_value) }}</strong></span>
              </div>
            </div>
          </div>
        </template>
      </template>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

const TREND_PRIORITY = { increasing: 0, stable: 1, decreasing: 2 }

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    const allForecasts = ref([])
    const inventoryItems = ref([])
    const budget = ref(50000)
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const submittedOrder = ref(null)

    const formatCurrency = (value) => {
      if (currentCurrency.value === 'JPY') {
        return '¥' + Math.round(value).toLocaleString()
      }
      return '$' + value.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })
    }

    const formatDate = (dateStr) => {
      const d = new Date(dateStr)
      if (isNaN(d.getTime())) return dateStr
      return d.toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' })
    }

    const matchedItems = computed(() => {
      const inventoryBySku = new Map(inventoryItems.value.map(inv => [inv.sku, inv]))
      return allForecasts.value
        .filter(f => inventoryBySku.has(f.item_sku))
        .map(f => ({
          item_sku: f.item_sku,
          item_name: f.item_name,
          trend: f.trend,
          forecasted_demand: f.forecasted_demand,
          unit_cost: inventoryBySku.get(f.item_sku).unit_cost
        }))
    })

    const recommendations = computed(() => {
      const sorted = [...matchedItems.value].sort(
        (a, b) => (TREND_PRIORITY[a.trend] ?? 3) - (TREND_PRIORITY[b.trend] ?? 3)
      )

      let running = 0
      return sorted.map(item => {
        const total_cost = item.forecasted_demand * item.unit_cost
        const inBudget = running + total_cost <= budget.value
        if (inBudget) running += total_cost
        return { ...item, total_cost, inBudget }
      })
    })

    const budgetRemaining = computed(() => {
      return budget.value - recommendations.value
        .filter(r => r.inBudget)
        .reduce((sum, r) => sum + r.total_cost, 0)
    })

    const inBudgetItems = computed(() => recommendations.value.filter(r => r.inBudget))

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
        error.value = 'Failed to load restocking data'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      try {
        const items = inBudgetItems.value.map(item => ({
          sku: item.item_sku,
          name: item.item_name,
          quantity: item.forecasted_demand,
          unit_price: item.unit_cost
        }))
        submittedOrder.value = await api.createRestockingOrder(items)
      } catch (err) {
        error.value = 'Failed to place order'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      t,
      allForecasts,
      inventoryItems,
      budget,
      loading,
      error,
      submitting,
      submittedOrder,
      recommendations,
      budgetRemaining,
      inBudgetItems,
      formatCurrency,
      formatDate,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding-bottom: 2rem;
}

.budget-label {
  display: block;
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.75rem;
}

.budget-display {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.75rem;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  margin-bottom: 0.625rem;
  cursor: pointer;
}

.budget-remaining {
  font-size: 0.875rem;
  color: #64748b;
}

.no-items {
  color: #64748b;
  font-size: 0.938rem;
  padding: 1.5rem 0;
}

.row-dimmed td {
  color: #94a3b8;
  opacity: 0.45;
}

.badge.over-budget {
  background: #f1f5f9;
  color: #64748b;
}

.order-actions {
  margin-top: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.place-order-btn {
  background: #2563eb;
  color: #ffffff;
  border: none;
  padding: 0.75rem 2rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
  align-self: flex-start;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-box {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  border-radius: 8px;
  padding: 1rem 1.25rem;
}

.success-title {
  font-weight: 700;
  font-size: 1rem;
  margin-bottom: 0.625rem;
}

.success-detail {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
  font-size: 0.875rem;
}
</style>
