<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Recommendations</h2>
      <p>Set a budget and review AI-ranked items to restock. Place an order when ready.</p>
    </div>

    <!-- Success Banner -->
    <div v-if="successMessage" class="success-banner">
      {{ successMessage }}
    </div>

    <!-- Budget Section -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Budget</h3>
      </div>

      <div class="budget-presets">
        <button
          v-for="preset in presets"
          :key="preset.value"
          class="preset-btn"
          :class="{ active: budget === preset.value }"
          @click="budget = preset.value"
        >
          {{ preset.label }}
        </button>
      </div>

      <div class="slider-row">
        <input
          type="range"
          min="0"
          max="50000"
          step="5000"
          v-model.number="budget"
          class="budget-slider"
        />
        <span class="budget-display">${{ budget.toLocaleString() }}</span>
      </div>

      <div v-if="recommendations" class="budget-summary">
        {{ recommendations.recommendations.length }} items fit within budget
        &middot; Total: ${{ recommendations.total_selected_cost.toLocaleString() }}
        &middot; Remaining: ${{ (budget - recommendations.total_selected_cost).toLocaleString() }}
      </div>
    </div>

    <!-- Recommendations Table -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Recommended Items</h3>
      </div>

      <div v-if="loading" class="loading">Loading recommendations...</div>
      <div v-else-if="recommendations">
        <!-- Items within budget -->
        <div v-if="recommendations.recommendations.length > 0">
          <div class="table-container">
            <table>
              <thead>
                <tr>
                  <th>Item Name</th>
                  <th>SKU</th>
                  <th>Trend</th>
                  <th>Qty to Order</th>
                  <th>Unit Cost</th>
                  <th>Total Cost</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="item in recommendations.recommendations" :key="item.item_sku">
                  <td>{{ item.item_name }}</td>
                  <td><code class="sku">{{ item.item_sku }}</code></td>
                  <td>
                    <span :class="['badge', trendClass(item.trend)]">{{ item.trend }}</span>
                  </td>
                  <td>{{ item.quantity }}</td>
                  <td>${{ item.unit_cost.toLocaleString() }}</td>
                  <td><strong>${{ item.total_cost.toLocaleString() }}</strong></td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
        <div v-else class="empty-state">
          No items fit within the current budget. Increase the budget to see recommendations.
        </div>

        <!-- Over-budget candidates -->
        <div v-if="recommendations.all_candidates && overBudgetItems.length > 0" class="over-budget-section">
          <div class="over-budget-header">Not included (over budget)</div>
          <div class="table-container">
            <table>
              <thead>
                <tr>
                  <th>Item Name</th>
                  <th>SKU</th>
                  <th>Trend</th>
                  <th>Qty to Order</th>
                  <th>Unit Cost</th>
                  <th>Total Cost</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="item in overBudgetItems" :key="item.item_sku" class="muted-row">
                  <td>{{ item.item_name }}</td>
                  <td><code class="sku">{{ item.item_sku }}</code></td>
                  <td>
                    <span :class="['badge', trendClass(item.trend)]">{{ item.trend }}</span>
                  </td>
                  <td>{{ item.quantity }}</td>
                  <td>${{ item.unit_cost.toLocaleString() }}</td>
                  <td>${{ item.total_cost.toLocaleString() }}</td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
      <div v-else class="loading">Loading...</div>
    </div>

    <!-- Delivery Option Selector -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Delivery Option</h3>
      </div>
      <div class="delivery-options">
        <div
          v-for="option in deliveryOptions"
          :key="option.key"
          class="delivery-card"
          :class="{ selected: selectedDelivery === option.key }"
          @click="selectedDelivery = option.key"
        >
          <div class="delivery-title">{{ option.label }}</div>
          <div class="delivery-days">{{ option.days }} days</div>
        </div>
      </div>
      <div class="expected-delivery">
        Expected by: {{ formatDeliveryDate(expectedDelivery) }}
      </div>
    </div>

    <!-- Place Order Button -->
    <div class="action-row">
      <button
        class="place-order-btn"
        :disabled="isOrderDisabled"
        @click="placeOrder"
      >
        Place Restocking Order
      </button>
      <span v-if="orderError" class="order-error">{{ orderError }}</span>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(25000)
    const recommendations = ref(null)
    const loading = ref(false)
    const selectedDelivery = ref('standard')
    const successMessage = ref('')
    const orderError = ref('')

    const presets = [
      { label: '$10K', value: 10000 },
      { label: '$25K', value: 25000 },
      { label: '$50K', value: 50000 }
    ]

    const deliveryOptions = [
      { key: 'express', label: 'Express', days: 7 },
      { key: 'standard', label: 'Standard', days: 14 },
      { key: 'economy', label: 'Economy', days: 21 }
    ]

    const leadDays = { express: 7, standard: 14, economy: 21 }

    const expectedDelivery = computed(() => {
      const d = new Date()
      d.setDate(d.getDate() + leadDays[selectedDelivery.value])
      return d.toISOString().split('T')[0]
    })

    const overBudgetItems = computed(() => {
      if (!recommendations.value || !recommendations.value.all_candidates) return []
      const includedSkus = new Set(recommendations.value.recommendations.map(i => i.item_sku))
      return recommendations.value.all_candidates.filter(i => !includedSkus.has(i.item_sku))
    })

    const isOrderDisabled = computed(() => {
      if (!recommendations.value) return true
      if (recommendations.value.recommendations.length === 0) return true
      if (!selectedDelivery.value) return true
      return false
    })

    const trendClass = (trend) => {
      if (!trend) return 'info'
      const t = trend.toLowerCase()
      if (t === 'increasing') return 'success'
      if (t === 'decreasing') return 'danger'
      return 'info'
    }

    const formatDeliveryDate = (dateStr) => {
      const d = new Date(dateStr)
      if (isNaN(d.getTime())) return dateStr
      return d.toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' })
    }

    let debounceTimer = null

    const loadRecommendations = async () => {
      loading.value = true
      try {
        const data = await api.getRestockRecommendations(budget.value)
        recommendations.value = data
      } catch (err) {
        console.error('Failed to load recommendations:', err)
        recommendations.value = null
      } finally {
        loading.value = false
      }
    }

    const debouncedLoad = () => {
      clearTimeout(debounceTimer)
      debounceTimer = setTimeout(loadRecommendations, 300)
    }

    watch(budget, debouncedLoad)

    const placeOrder = async () => {
      orderError.value = ''
      if (isOrderDisabled.value) return
      try {
        const result = await api.placeRestockOrder({
          items: recommendations.value.recommendations,
          total_cost: recommendations.value.total_selected_cost,
          delivery_option: selectedDelivery.value,
          expected_delivery: expectedDelivery.value
        })
        successMessage.value = `Restocking order ${result.order_number} placed successfully. View it in the Orders tab.`
        selectedDelivery.value = 'standard'
        recommendations.value = null
        await loadRecommendations()
      } catch (err) {
        orderError.value = 'Failed to place order. Please try again.'
        console.error(err)
      }
    }

    onMounted(loadRecommendations)

    return {
      budget,
      recommendations,
      loading,
      selectedDelivery,
      successMessage,
      orderError,
      presets,
      deliveryOptions,
      expectedDelivery,
      overBudgetItems,
      isOrderDisabled,
      trendClass,
      formatDeliveryDate,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}

.budget-presets {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.preset-btn {
  padding: 0.5rem 1.25rem;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  background: white;
  color: #475569;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.15s ease;
}

.preset-btn:hover {
  border-color: #2563eb;
  color: #2563eb;
  background: #eff6ff;
}

.preset-btn.active {
  border-color: #2563eb;
  background: #2563eb;
  color: white;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1.25rem;
  margin-bottom: 0.875rem;
}

.budget-slider {
  flex: 1;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-display {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 110px;
  text-align: right;
}

.budget-summary {
  color: #64748b;
  font-size: 0.875rem;
}

.sku {
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  font-size: 0.813rem;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  color: #475569;
}

.empty-state {
  text-align: center;
  padding: 2rem;
  color: #64748b;
  font-size: 0.938rem;
}

.over-budget-section {
  margin-top: 1.5rem;
  border-top: 1px solid #e2e8f0;
  padding-top: 1rem;
}

.over-budget-header {
  font-size: 0.813rem;
  font-weight: 600;
  color: #94a3b8;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.75rem;
}

.muted-row td {
  opacity: 0.5;
}

.delivery-options {
  display: flex;
  gap: 1rem;
  margin-bottom: 1rem;
}

.delivery-card {
  flex: 1;
  border: 2px solid #e2e8f0;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  cursor: pointer;
  transition: all 0.15s ease;
  background: white;
}

.delivery-card:hover {
  border-color: #93c5fd;
}

.delivery-card.selected {
  border-color: #2563eb;
  background: #eff6ff;
}

.delivery-title {
  font-size: 1rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.25rem;
}

.delivery-days {
  font-size: 0.875rem;
  color: #64748b;
}

.delivery-card.selected .delivery-title {
  color: #1d4ed8;
}

.expected-delivery {
  font-size: 0.875rem;
  color: #64748b;
}

.action-row {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding-bottom: 2rem;
}

.place-order-btn {
  padding: 0.75rem 2rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  color: #94a3b8;
  cursor: not-allowed;
}

.order-error {
  color: #dc2626;
  font-size: 0.875rem;
}
</style>
