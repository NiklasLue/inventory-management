<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="lastOrder" class="success-banner">
      <div class="success-text">
        <strong>{{ t('restocking.successTitle') }}</strong>
        <span>{{ t('restocking.successDetail', { orderNumber: lastOrder.order_number }) }}</span>
      </div>
      <router-link to="/orders" class="success-link">{{ t('restocking.viewOrders') }}</router-link>
    </div>

    <div class="card budget-card">
      <div class="card-header">
        <h3 class="card-title">{{ t('restocking.budgetLabel') }}</h3>
        <div class="budget-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</div>
      </div>
      <div class="budget-controls">
        <input
          type="range"
          :min="BUDGET_MIN"
          :max="BUDGET_MAX"
          :step="BUDGET_STEP"
          v-model.number="budget"
          class="budget-slider"
        />
        <input
          type="number"
          :min="BUDGET_MIN"
          :max="BUDGET_MAX"
          :step="BUDGET_STEP"
          v-model.number="budget"
          class="budget-number"
        />
      </div>
      <div class="budget-marks">
        <span>{{ currencySymbol }}{{ BUDGET_MIN.toLocaleString() }}</span>
        <span>{{ currencySymbol }}{{ BUDGET_MAX.toLocaleString() }}</span>
      </div>
    </div>

    <div class="stats-grid">
      <div class="stat-card info">
        <div class="stat-label">{{ t('restocking.budgetTotal') }}</div>
        <div class="stat-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</div>
      </div>
      <div class="stat-card warning">
        <div class="stat-label">{{ t('restocking.budgetUsed') }}</div>
        <div class="stat-value">{{ currencySymbol }}{{ budgetUsed.toLocaleString() }}</div>
      </div>
      <div class="stat-card success">
        <div class="stat-label">{{ t('restocking.budgetRemaining') }}</div>
        <div class="stat-value">{{ currencySymbol }}{{ remaining.toLocaleString() }}</div>
      </div>
      <div class="stat-card">
        <div class="stat-label">{{ t('restocking.itemCount') }}</div>
        <div class="stat-value">{{ recommendations.length }}</div>
      </div>
    </div>

    <div class="card">
      <div class="card-header">
        <h3 class="card-title">{{ t('restocking.recommendations') }} ({{ recommendations.length }})</h3>
        <button
          class="place-order-btn"
          :disabled="recommendations.length === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? t('restocking.submitting') : t('restocking.placeOrder') }}
        </button>
      </div>

      <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else-if="recommendations.length === 0" class="empty">
        {{ t('restocking.empty') }}
      </div>
      <div v-else class="table-container">
        <table>
          <thead>
            <tr>
              <th>{{ t('restocking.table.sku') }}</th>
              <th>{{ t('restocking.table.itemName') }}</th>
              <th>{{ t('restocking.table.category') }}</th>
              <th>{{ t('restocking.table.warehouse') }}</th>
              <th class="num">{{ t('restocking.table.currentStock') }}</th>
              <th class="num">{{ t('restocking.table.forecastedDemand') }}</th>
              <th class="num">{{ t('restocking.table.shortfall') }}</th>
              <th class="num">{{ t('restocking.table.unitCost') }}</th>
              <th class="num">{{ t('restocking.table.suggestedQty') }}</th>
              <th class="num">{{ t('restocking.table.lineTotal') }}</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="rec in recommendations" :key="rec.item_sku">
              <td><strong>{{ rec.item_sku }}</strong></td>
              <td>{{ rec.item_name }}</td>
              <td>{{ rec.category }}</td>
              <td>{{ rec.warehouse }}</td>
              <td class="num">{{ rec.current_stock }}</td>
              <td class="num">{{ rec.forecasted_demand }}</td>
              <td class="num shortfall">{{ rec.shortfall }}</td>
              <td class="num">{{ currencySymbol }}{{ rec.unit_cost.toLocaleString() }}</td>
              <td class="num"><strong>{{ rec.suggested_qty }}</strong></td>
              <td class="num"><strong>{{ currencySymbol }}{{ rec.line_total.toLocaleString() }}</strong></td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

const BUDGET_MIN = 1000
const BUDGET_MAX = 500000
const BUDGET_STEP = 1000
const DEBOUNCE_MS = 300

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()
    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const budget = ref(50000)
    const recommendations = ref([])
    const budgetUsed = ref(0)
    const loading = ref(false)
    const submitting = ref(false)
    const error = ref(null)
    const lastOrder = ref(null)

    const remaining = computed(() => Math.max(0, budget.value - budgetUsed.value))

    const loadRecommendations = async () => {
      if (budget.value < BUDGET_MIN) return
      try {
        loading.value = true
        error.value = null
        const data = await api.getRestockingRecommendations(budget.value)
        recommendations.value = data.recommendations
        budgetUsed.value = data.budget_used
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
        recommendations.value = []
        budgetUsed.value = 0
      } finally {
        loading.value = false
      }
    }

    // Debounce reloads so dragging the slider doesn't fire a request per pixel.
    let debounceTimer = null
    watch(budget, () => {
      if (debounceTimer) clearTimeout(debounceTimer)
      debounceTimer = setTimeout(loadRecommendations, DEBOUNCE_MS)
    })

    const placeOrder = async () => {
      if (recommendations.value.length === 0 || submitting.value) return
      try {
        submitting.value = true
        error.value = null
        const payload = {
          items: recommendations.value.map(r => ({
            item_sku: r.item_sku,
            item_name: r.item_name,
            quantity: r.suggested_qty,
            unit_cost: r.unit_cost,
            category: r.category,
            warehouse: r.warehouse
          }))
        }
        const created = await api.submitRestockingOrder(payload)
        lastOrder.value = created
        // Reset recommendations so the user doesn't double-submit the same set.
        recommendations.value = []
        budgetUsed.value = 0
      } catch (err) {
        error.value = 'Failed to submit order: ' + (err.response?.data?.detail || err.message)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      t,
      budget,
      recommendations,
      budgetUsed,
      remaining,
      loading,
      submitting,
      error,
      lastOrder,
      currencySymbol,
      placeOrder,
      BUDGET_MIN,
      BUDGET_MAX,
      BUDGET_STEP
    }
  }
}
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.25rem;
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

.budget-controls {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-top: 0.5rem;
}

.budget-slider {
  flex: 1;
  -webkit-appearance: none;
  appearance: none;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.budget-number {
  width: 140px;
  padding: 0.5rem 0.75rem;
  border: 1px solid #cbd5e1;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  color: #0f172a;
  text-align: right;
}

.budget-number:focus {
  outline: none;
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.budget-marks {
  display: flex;
  justify-content: space-between;
  margin-top: 0.5rem;
  font-size: 0.75rem;
  color: #64748b;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}

.empty {
  padding: 3rem 1rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.num {
  text-align: right;
}

.shortfall {
  color: #dc2626;
  font-weight: 600;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.25rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}

.success-text {
  display: flex;
  flex-direction: column;
  gap: 0.125rem;
  color: #065f46;
  font-size: 0.938rem;
}

.success-text strong {
  font-size: 1rem;
}

.success-link {
  color: #065f46;
  text-decoration: underline;
  font-weight: 600;
  font-size: 0.875rem;
  white-space: nowrap;
}
</style>
