<template>
  <div class="restocking">
    <!-- page-header -->
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <!-- loading / error states -->
    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>

    <!-- main content -->
    <div v-else>
      <!-- budget card -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetLabel') }}</h3>
          <span class="budget-display">{{ formatCurrency(budget) }}</span>
        </div>
        <input
          type="range"
          v-model.number="budget"
          min="0"
          max="100000"
          step="1000"
          class="budget-slider"
        />
        <div class="slider-labels">
          <span>$0</span>
          <span>$100K</span>
        </div>
      </div>

      <!-- stats grid -->
      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.itemsRecommended') }}</div>
          <div class="stat-value">{{ selectedItems.length }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('restocking.totalCost') }}</div>
          <div class="stat-value cost-value">{{ formatCurrency(totalCost) }}</div>
        </div>
        <div class="stat-card" :class="remainingBudget >= 0 ? 'success' : 'danger'">
          <div class="stat-label">{{ t('restocking.remainingBudget') }}</div>
          <div class="stat-value">{{ formatCurrency(remainingBudget) }}</div>
        </div>
      </div>

      <!-- success/error banners -->
      <div v-if="submitSuccess" class="alert alert-success">{{ t('restocking.orderSuccess') }}</div>
      <div v-if="submitError" class="alert alert-error">{{ submitError }}</div>

      <!-- recommendations card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.title') }}</h3>
          <button
            class="btn-primary"
            @click="placeOrder"
            :disabled="!canPlaceOrder"
          >{{ t('restocking.placeOrder') }}</button>
        </div>

        <!-- empty state -->
        <div v-if="selectedItems.length === 0" class="empty-state">
          {{ t('restocking.noItems') }}
        </div>

        <!-- table -->
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.forecastedDemand') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.totalCost') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in selectedItems" :key="item.sku">
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td>{{ item.quantity.toLocaleString() }}</td>
                <td>{{ formatCurrency(item.unit_price) }}</td>
                <td><strong>{{ formatCurrency(item.total_cost) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t } = useI18n()

    // Refs
    const budget = ref(50000)
    const allForecasts = ref([])
    const inventoryItems = ref([])
    const loading = ref(true)
    const submitting = ref(false)
    const error = ref(null)
    const submitSuccess = ref(false)
    const submitError = ref(null)

    // Computed: Map of SKU -> unit_cost from inventory
    const inventoryBySku = computed(() => {
      const map = new Map()
      for (const item of inventoryItems.value) {
        map.set(item.sku, item.unit_cost)
      }
      return map
    })

    // Computed: Forecasts with increasing trend, sorted by forecasted_demand descending
    const increasingForecasts = computed(() => {
      return allForecasts.value
        .filter(f => f.trend === 'increasing')
        .slice()
        .sort((a, b) => b.forecasted_demand - a.forecasted_demand)
    })

    // Computed: Greedy selection within budget
    const selectedItems = computed(() => {
      let remaining = budget.value
      const selected = []

      for (const forecast of increasingForecasts.value) {
        const unitCost = inventoryBySku.value.get(forecast.item_sku)
        if (unitCost === undefined) continue

        const cost = unitCost * forecast.forecasted_demand
        if (cost <= remaining) {
          selected.push({
            sku: forecast.item_sku,
            name: forecast.item_name,
            quantity: forecast.forecasted_demand,
            unit_price: unitCost,
            total_cost: cost
          })
          remaining -= cost
        }
      }

      return selected
    })

    // Computed: Sum of total_cost across selected items
    const totalCost = computed(() => {
      return selectedItems.value.reduce((s, i) => s + i.total_cost, 0)
    })

    // Computed: Budget minus total cost
    const remainingBudget = computed(() => {
      return budget.value - totalCost.value
    })

    // Computed: Whether order can be placed
    const canPlaceOrder = computed(() => {
      return selectedItems.value.length > 0 && !submitting.value
    })

    // Methods
    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])
        allForecasts.value = forecasts
        inventoryItems.value = inventory
      } catch (err) {
        error.value = 'Failed to load data'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      submitSuccess.value = false
      submitError.value = null
      try {
        await api.createRestockingOrder({
          items: selectedItems.value,
          total_value: totalCost.value
        })
        submitSuccess.value = true
      } catch (err) {
        submitError.value = 'Failed to submit order. Please try again.'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    const formatCurrency = (val) => {
      return val.toLocaleString('en-US', { style: 'currency', currency: 'USD' })
    }

    // Lifecycle
    onMounted(loadData)

    return {
      t,
      budget,
      allForecasts,
      inventoryItems,
      loading,
      submitting,
      error,
      submitSuccess,
      submitError,
      inventoryBySku,
      increasingForecasts,
      selectedItems,
      totalCost,
      remainingBudget,
      canPlaceOrder,
      loadData,
      placeOrder,
      formatCurrency
    }
  }
}
</script>

<style scoped>
.budget-card .card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
}

.budget-slider {
  width: 100%;
  margin: 1rem 0 0.5rem;
  accent-color: #2563eb;
  cursor: pointer;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #64748b;
}

.cost-value {
  color: #0f172a;
}

.empty-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.alert {
  padding: 0.875rem 1rem;
  border-radius: 8px;
  margin-bottom: 1rem;
  font-size: 0.875rem;
  font-weight: 500;
}

.alert-success {
  background: #d1fae5;
  color: #065f46;
  border: 1px solid #6ee7b7;
}

.alert-error {
  background: #fef2f2;
  color: #991b1b;
  border: 1px solid #fecaca;
}

.btn-primary {
  padding: 0.5rem 1.25rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #93c5fd;
  cursor: not-allowed;
}
</style>
