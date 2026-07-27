<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Slider -->
      <div class="card budget-card">
        <div class="budget-header">
          <span class="budget-label">{{ t('restocking.budget') }}</span>
          <span class="budget-amount">{{ formatCurrency(budget) }}</span>
        </div>
        <input
          type="range"
          min="1000"
          max="50000"
          step="500"
          v-model.number="budget"
          class="budget-slider"
        />
        <div class="budget-range-labels">
          <span>{{ formatCurrency(1000) }}</span>
          <span>{{ formatCurrency(50000) }}</span>
        </div>
      </div>

      <!-- Summary Stats -->
      <div class="stats-grid">
        <div class="stat-card warning">
          <div class="stat-label">{{ t('restocking.itemsToRestock') }}</div>
          <div class="stat-value">{{ selectedItemsCount }}</div>
        </div>
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.estimatedCost') }}</div>
          <div class="stat-value">{{ formatCurrency(selectedTotal) }}</div>
        </div>
        <div :class="['stat-card', remainingBudget >= 0 ? 'success' : 'danger']">
          <div class="stat-label">{{ t('restocking.remainingBudget') }}</div>
          <div class="stat-value">{{ formatCurrency(Math.abs(remainingBudget)) }}</div>
        </div>
      </div>

      <!-- No Items State -->
      <div v-if="suggestions.length === 0" class="card no-items-card">
        <p class="no-items-message">{{ t('restocking.noItems') }}</p>
      </div>

      <!-- Recommendations Table + Place Order -->
      <div v-else>
        <!-- Success Message (replaces table/button after order placed) -->
        <div v-if="orderResult" class="card success-card">
          <div class="success-content">
            <div class="success-icon">&#10003;</div>
            <div class="success-details">
              <p class="success-title">{{ t('restocking.orderSuccess') }}</p>
              <p class="success-order-number">
                {{ t('restocking.orderNumber') }}: <strong>{{ orderResult.order_number }}</strong>
              </p>
              <router-link to="/orders" class="view-orders-link">
                {{ t('restocking.viewOrders') }}
              </router-link>
            </div>
          </div>
        </div>

        <div v-else>
          <div class="card">
            <div class="card-header">
              <h3 class="card-title">{{ t('restocking.title') }} ({{ suggestions.length }})</h3>
            </div>
            <div class="table-container">
              <table>
                <thead>
                  <tr>
                    <th>{{ t('restocking.table.select') }}</th>
                    <th>{{ t('restocking.table.sku') }}</th>
                    <th>{{ t('restocking.table.itemName') }}</th>
                    <th>{{ t('restocking.table.warehouse') }}</th>
                    <th>{{ t('restocking.table.onHand') }}</th>
                    <th>{{ t('restocking.table.reorderPoint') }}</th>
                    <th>{{ t('restocking.table.suggestedQty') }}</th>
                    <th>{{ t('restocking.table.unitCost') }}</th>
                    <th>{{ t('restocking.table.lineTotal') }}</th>
                    <th>{{ t('restocking.table.priority') }}</th>
                  </tr>
                </thead>
                <tbody>
                  <tr
                    v-for="item in suggestions"
                    :key="item.sku"
                    :class="{ 'row-dimmed': isRowDimmed(item) }"
                  >
                    <td>
                      <input
                        type="checkbox"
                        :checked="itemState[item.sku] && itemState[item.sku].selected"
                        :disabled="isRowDimmed(item)"
                        @change="toggleItem(item.sku)"
                      />
                    </td>
                    <td><strong>{{ item.sku }}</strong></td>
                    <td>{{ translateProductName(item.name) }}</td>
                    <td>{{ item.warehouse }}</td>
                    <td>{{ item.quantity_on_hand }}</td>
                    <td>{{ item.reorder_point }}</td>
                    <td>
                      <input
                        type="number"
                        min="1"
                        class="qty-input"
                        :value="itemState[item.sku] ? itemState[item.sku].quantity : item.suggested_quantity"
                        @input="updateQuantity(item.sku, $event)"
                      />
                    </td>
                    <td>{{ formatCurrency(item.unit_cost) }}</td>
                    <td>{{ formatCurrency(getLineTotal(item)) }}</td>
                    <td>
                      <span :class="['badge', getPriorityBadgeClass(item.priority)]">
                        {{ t(`priority.${item.priority}`) }}
                      </span>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>

          <button
            class="place-order-btn"
            :disabled="!canPlaceOrder"
            @click="placeOrder"
          >
            {{ submitting ? 'Submitting...' : t('restocking.placeOrder') }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'
import { formatCurrency as formatCurrencyUtil } from '../utils/currency'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, translateProductName } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const suggestions = ref([])
    const itemState = ref({})
    const budget = ref(10000)
    const submitting = ref(false)
    const orderResult = ref(null)

    const formatCurrency = (value) => {
      return formatCurrencyUtil(value, currentCurrency.value)
    }

    const loadSuggestions = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockingSuggestions()
        suggestions.value = data
        const state = {}
        data.forEach(item => {
          state[item.sku] = { selected: true, quantity: item.suggested_quantity }
        })
        itemState.value = state
      } catch (err) {
        error.value = 'Failed to load restocking suggestions'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const getLineTotal = (item) => {
      const state = itemState.value[item.sku]
      const qty = state ? state.quantity : item.suggested_quantity
      return qty * item.unit_cost
    }

    const selectedItemsCount = computed(() => {
      return suggestions.value.filter(item => {
        const state = itemState.value[item.sku]
        return state && state.selected
      }).length
    })

    const selectedTotal = computed(() => {
      return suggestions.value.reduce((sum, item) => {
        const state = itemState.value[item.sku]
        if (state && state.selected) {
          return sum + getLineTotal(item)
        }
        return sum
      }, 0)
    })

    const remainingBudget = computed(() => budget.value - selectedTotal.value)

    const canPlaceOrder = computed(() => {
      return selectedItemsCount.value > 0 && selectedTotal.value <= budget.value && !submitting.value
    })

    const isRowDimmed = (item) => {
      const state = itemState.value[item.sku]
      if (state && state.selected) return false
      return getLineTotal(item) > remainingBudget.value
    }

    const toggleItem = (sku) => {
      if (itemState.value[sku]) {
        itemState.value[sku].selected = !itemState.value[sku].selected
      }
    }

    const updateQuantity = (sku, event) => {
      const val = parseInt(event.target.value, 10)
      if (!isNaN(val) && val >= 1 && itemState.value[sku]) {
        itemState.value[sku].quantity = val
      }
    }

    const getPriorityBadgeClass = (priority) => {
      const map = { high: 'danger', medium: 'warning', low: 'info' }
      return map[priority] || priority
    }

    const placeOrder = async () => {
      if (!canPlaceOrder.value) return
      submitting.value = true
      error.value = null
      try {
        const selectedItemsList = suggestions.value
          .filter(item => itemState.value[item.sku] && itemState.value[item.sku].selected)
          .map(item => ({
            sku: item.sku,
            name: item.name,
            quantity: itemState.value[item.sku].quantity,
            unit_cost: item.unit_cost
          }))
        const result = await api.submitRestockingOrder({
          items: selectedItemsList,
          total_budget: budget.value
        })
        orderResult.value = result
      } catch (err) {
        error.value = 'Failed to place restocking order'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadSuggestions)

    return {
      t,
      loading,
      error,
      suggestions,
      itemState,
      budget,
      submitting,
      orderResult,
      formatCurrency,
      translateProductName,
      selectedItemsCount,
      selectedTotal,
      remainingBudget,
      canPlaceOrder,
      getLineTotal,
      isRowDimmed,
      toggleItem,
      updateQuantity,
      getPriorityBadgeClass,
      placeOrder
    }
  }
}
</script>

<style scoped>
/* Budget Card */
.budget-card {
  margin-bottom: 1.25rem;
}

.budget-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.budget-amount {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  height: 6px;
  -webkit-appearance: none;
  appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  background: #2563eb;
  border-radius: 50%;
  cursor: pointer;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  transition: background 0.2s;
}

.budget-slider::-webkit-slider-thumb:hover {
  background: #1d4ed8;
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  background: #2563eb;
  border: none;
  border-radius: 50%;
  cursor: pointer;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  margin-top: 0.5rem;
}

.budget-range-labels span {
  font-size: 0.75rem;
  color: #94a3b8;
}

/* Quantity Input */
.qty-input {
  width: 80px;
  text-align: right;
  padding: 0.25rem 0.375rem;
  border: 1px solid #e2e8f0;
  border-radius: 4px;
  font-size: 0.875rem;
  color: #334155;
  background: #f8fafc;
}

.qty-input:focus {
  outline: none;
  border-color: #2563eb;
  background: white;
}

/* Dimmed rows for items exceeding remaining budget */
.row-dimmed {
  opacity: 0.5;
}

/* Place Order Button */
.place-order-btn {
  display: block;
  width: 100%;
  padding: 0.875rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
  margin-bottom: 1.25rem;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

/* No Items State */
.no-items-card {
  text-align: center;
  padding: 3rem;
}

.no-items-message {
  font-size: 0.938rem;
  color: #64748b;
}

/* Success Card */
.success-card {
  border-left: 4px solid #10b981;
  background: #f0fdf4;
}

.success-content {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  padding: 0.5rem;
}

.success-icon {
  width: 48px;
  height: 48px;
  background: #d1fae5;
  color: #059669;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.5rem;
  font-weight: 700;
  flex-shrink: 0;
}

.success-details {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.success-title {
  font-size: 1.125rem;
  font-weight: 600;
  color: #065f46;
}

.success-order-number {
  font-size: 0.875rem;
  color: #334155;
}

.view-orders-link {
  display: inline-block;
  margin-top: 0.25rem;
  color: #2563eb;
  font-size: 0.875rem;
  font-weight: 500;
  text-decoration: none;
}

.view-orders-link:hover {
  text-decoration: underline;
}
</style>
