<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Manage inventory restocking based on budget constraints</p>
    </div>

    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Budget</h3>
      </div>
      <div class="budget-slider-container">
        <div class="slider-row">
          <span class="slider-min">{{ currencySymbol }}0</span>
          <input
            type="range"
            class="budget-slider"
            min="0"
            max="50000"
            step="500"
            v-model.number="budget"
          />
          <span class="slider-max">{{ currencySymbol }}50,000</span>
          <span class="budget-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
        </div>
      </div>
    </div>

    <div v-if="successMessage" class="success-banner">{{ successMessage }}</div>
    <div v-if="orderError" class="error">{{ orderError }}</div>

    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Recommendations</h3>
        <button
          class="btn-place-order"
          :disabled="recommendations.length === 0 || placingOrder"
          @click="placeOrder"
        >
          {{ placingOrder ? 'Placing Order...' : 'Place Order' }}
        </button>
      </div>

      <div v-if="loadingRecommendations" class="loading">Loading recommendations...</div>
      <div v-else-if="recommendationsError" class="error">{{ recommendationsError }}</div>
      <template v-else>
        <div v-if="recommendations.length > 0" class="summary-bar">
          {{ recommendations.length }} items recommended
          &middot; Total cost: {{ currencySymbol }}{{ totalCost.toLocaleString() }}
          &middot; Remaining budget: {{ currencySymbol }}{{ remainingBudget.toLocaleString() }}
        </div>

        <div class="table-container">
          <div v-if="recommendations.length === 0" class="empty-state">
            No items need restocking within this budget
          </div>
          <table v-else>
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Category</th>
                <th>Warehouse</th>
                <th>On Hand</th>
                <th>Reorder Point</th>
                <th>Restock Qty</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.sku">
                <td><strong>{{ item.name }}</strong></td>
                <td>{{ item.sku }}</td>
                <td>{{ item.category }}</td>
                <td>{{ item.warehouse }}</td>
                <td>{{ item.quantity_on_hand }}</td>
                <td>{{ item.reorder_point }}</td>
                <td>{{ item.restock_quantity }}</td>
                <td>{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td>{{ currencySymbol }}{{ item.total_cost.toLocaleString() }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </template>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const budget = ref(10000)
    const recommendations = ref([])
    const loadingRecommendations = ref(false)
    const recommendationsError = ref(null)
    const placingOrder = ref(false)
    const successMessage = ref(null)
    const orderError = ref(null)

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, item) => sum + item.total_cost, 0)
    )

    const remainingBudget = computed(() => budget.value - totalCost.value)

    const loadRecommendations = async () => {
      loadingRecommendations.value = true
      recommendationsError.value = null
      try {
        recommendations.value = await api.getRestockingRecommendations(budget.value)
      } catch (err) {
        recommendationsError.value = 'Failed to load recommendations'
        console.error(err)
      } finally {
        loadingRecommendations.value = false
      }
    }

    let debounceTimer = null

    watch(budget, () => {
      clearTimeout(debounceTimer)
      debounceTimer = setTimeout(() => {
        loadRecommendations()
      }, 300)
    })

    const placeOrder = async () => {
      if (recommendations.value.length === 0) return

      placingOrder.value = true
      successMessage.value = null
      orderError.value = null

      const items = recommendations.value.map(item => ({
        sku: item.sku,
        name: item.name,
        quantity: item.restock_quantity,
        unit_cost: item.unit_cost
      }))

      try {
        const result = await api.placeRestockingOrder(items)
        successMessage.value = `Restocking order ${result.order_number} placed successfully. Expected delivery: ${formatDate(result.expected_delivery)}`
        recommendations.value = []
      } catch (err) {
        orderError.value = 'Failed to place restocking order. Please try again.'
        console.error(err)
      } finally {
        placingOrder.value = false
      }
    }

    const formatDate = (dateString) => {
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      return date.toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'short',
        day: 'numeric'
      })
    }

    onMounted(() => loadRecommendations())

    return {
      currencySymbol,
      budget,
      recommendations,
      loadingRecommendations,
      recommendationsError,
      placingOrder,
      successMessage,
      orderError,
      totalCost,
      remainingBudget,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-slider-container {
  padding: 0.5rem 0 0.75rem;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.slider-min,
.slider-max {
  font-size: 0.813rem;
  color: #64748b;
  white-space: nowrap;
}

.budget-slider {
  flex: 1;
  height: 6px;
  appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
}

.budget-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
}

.budget-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 90px;
  text-align: right;
}

.summary-bar {
  background: #f1f5f9;
  border-radius: 6px;
  padding: 0.625rem 1rem;
  font-size: 0.875rem;
  color: #475569;
  font-weight: 500;
  margin-bottom: 1rem;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

.btn-place-order {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 0.5rem 1.25rem;
  font-size: 0.875rem;
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

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}
</style>
