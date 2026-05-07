<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Success state -->
      <div v-if="successOrder" class="success-banner">
        <div>
          <strong>{{ successOrder.order_number }}</strong> — {{ t('restocking.successToast') }}
        </div>
        <button @click="goToOrders" class="btn-primary">{{ t('restocking.viewInOrders') }}</button>
      </div>

      <!-- Budget + Lead time controls (always visible once data loads) -->
      <div class="card">
        <div class="controls-grid">
          <div class="control-block">
            <label class="control-label">
              {{ t('restocking.budget') }}: <strong>{{ formatCurrencyDisplay(budget) }}</strong>
              <span class="muted">/ {{ formatCurrencyDisplay(maxBudget) }}</span>
            </label>
            <input
              type="range"
              :min="0"
              :max="maxBudget"
              :step="1000"
              v-model.number="budget"
              class="budget-slider"
            />
          </div>
          <div class="control-block">
            <label class="control-label">{{ t('restocking.leadTime') }}</label>
            <div class="segmented">
              <button
                v-for="opt in [7, 14, 21, 30]"
                :key="opt"
                :class="['seg-btn', { active: leadTime === opt }]"
                @click="leadTime = opt"
              >{{ opt }} {{ t('restocking.days') }}</button>
            </div>
          </div>
        </div>
      </div>

      <!-- Recommendations table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }} ({{ recommendations.length }})</h3>
        </div>

        <!-- Empty state: truly nothing to restock across all lead times -->
        <div v-if="candidates.length === 0 && leadTime === 30" style="text-align: center; padding: 2rem; color: #64748b;">
          {{ t('restocking.nothingToRestock') }}
        </div>

        <!-- Empty state: no items qualify for the selected lead time -->
        <div v-else-if="candidates.length === 0" style="text-align: center; padding: 2rem; color: #64748b;">
          {{ t('restocking.noItemsForLeadTime') }}
        </div>

        <!-- Table when there are candidates -->
        <template v-else>
          <div class="table-container">
            <table>
              <thead>
                <tr>
                  <th>{{ t('restocking.table.sku') }}</th>
                  <th>{{ t('restocking.table.itemName') }}</th>
                  <th>{{ t('restocking.table.trend') }}</th>
                  <th>{{ t('restocking.table.shortage') }}</th>
                  <th>{{ t('restocking.table.orderQty') }}</th>
                  <th>{{ t('restocking.table.unitCost') }}</th>
                  <th>{{ t('restocking.table.subtotal') }}</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="r in recommendations" :key="r.sku">
                  <td><strong>{{ r.sku }}</strong></td>
                  <td>{{ translateProductName(r.name) }}</td>
                  <td><span :class="['badge', r.trend]">{{ t('trends.' + r.trend) }}</span></td>
                  <td>{{ r.shortage }}</td>
                  <td>{{ r.quantity }}</td>
                  <td>{{ formatCurrencyDisplay(r.unit_cost) }}</td>
                  <td><strong>{{ formatCurrencyDisplay(r.subtotal) }}</strong></td>
                </tr>
                <tr v-if="recommendations.length === 0">
                  <td colspan="7" style="text-align: center; padding: 1.5rem; color: #64748b;">
                    {{ t('common.noData') }}
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
          <div class="footer-bar">
            <div class="totals">
              <span>{{ t('restocking.totalSpend') }}: <strong>{{ formatCurrencyDisplay(totalSpend) }}</strong></span>
              <span class="muted">{{ t('restocking.remaining') }}: {{ formatCurrencyDisplay(remaining) }}</span>
            </div>
            <button
              class="btn-primary"
              :disabled="recommendations.length === 0 || submitting"
              @click="placeOrder"
            >
              {{ submitting ? t('restocking.placing') : t('restocking.placeOrder') }}
            </button>
          </div>
        </template>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'
import { formatCurrency } from '../utils/currency'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, translateProductName } = useI18n()
    const router = useRouter()

    // State
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const successOrder = ref(null)

    const demandForecasts = ref([])
    const inventoryItems = ref([])

    const budget = ref(0)
    const leadTime = ref(14)

    // Currency formatter that reacts to locale switch
    const formatCurrencyDisplay = (v) => formatCurrency(v, currentCurrency.value)

    // Load data
    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [demands, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])
        demandForecasts.value = demands
        inventoryItems.value = inventory
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // Build candidate list: demand items that have a shortage
    const candidates = computed(() => {
      const trendOrder = { increasing: 0, stable: 1, decreasing: 2 }

      const result = []
      for (const demand of demandForecasts.value) {
        const inventory = inventoryItems.value.find(inv => inv.sku === demand.item_sku)
        if (!inventory) continue

        // Only include items whose supplier lead time fits within the selected threshold
        if ((inventory.supplier_lead_days ?? 14) > leadTime.value) continue

        const shortage = Math.max(0, demand.forecasted_demand - inventory.quantity_on_hand)
        if (shortage === 0) continue

        const restock_cost = shortage * inventory.unit_cost
        result.push({
          sku: demand.item_sku,
          name: inventory.name,
          trend: demand.trend,
          shortage,
          unit_cost: inventory.unit_cost,
          restock_cost
        })
      }

      // Sort: increasing first, then stable, then decreasing; within bucket sort by shortage desc
      result.sort((a, b) => {
        const trendDiff = (trendOrder[a.trend] ?? 1) - (trendOrder[b.trend] ?? 1)
        if (trendDiff !== 0) return trendDiff
        return b.shortage - a.shortage
      })

      return result
    })

    const maxBudget = computed(() => {
      const total = candidates.value.reduce((sum, c) => sum + c.restock_cost, 0)
      if (total === 0) return 0
      return Math.ceil(total / 1000) * 1000
    })

    // Clamp budget when lead-time change shrinks the pool of eligible items
    watch(maxBudget, (newMax) => {
      if (budget.value > newMax) budget.value = newMax
    })

    // Greedy pack within budget
    const recommendations = computed(() => {
      if (budget.value <= 0) return []

      const result = []
      let remaining = budget.value

      for (const candidate of candidates.value) {
        if (remaining <= 0) break
        if (candidate.unit_cost > remaining) continue

        const maxQty = Math.floor(remaining / candidate.unit_cost)
        const quantity = Math.min(candidate.shortage, maxQty)
        if (quantity <= 0) continue

        const subtotal = quantity * candidate.unit_cost
        result.push({
          sku: candidate.sku,
          name: candidate.name,
          trend: candidate.trend,
          shortage: candidate.shortage,
          quantity,
          unit_cost: candidate.unit_cost,
          subtotal
        })
        remaining -= subtotal
      }

      return result
    })

    const totalSpend = computed(() => {
      return recommendations.value.reduce((sum, r) => sum + r.subtotal, 0)
    })

    const remaining = computed(() => {
      return budget.value - totalSpend.value
    })

    // Place the restock order
    const placeOrder = async () => {
      if (recommendations.value.length === 0 || submitting.value) return

      submitting.value = true
      error.value = null
      try {
        const response = await api.createRestockOrder({
          items: recommendations.value.map(r => ({
            sku: r.sku,
            name: r.name,
            quantity: r.quantity,
            unit_price: r.unit_cost
          })),
          lead_time_days: leadTime.value
        })
        successOrder.value = response
        budget.value = 0
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    const goToOrders = () => {
      router.push('/orders')
    }

    onMounted(loadData)

    return {
      t,
      loading,
      error,
      submitting,
      successOrder,
      budget,
      leadTime,
      candidates,
      maxBudget,
      recommendations,
      totalSpend,
      remaining,
      formatCurrencyDisplay,
      translateProductName,
      placeOrder,
      goToOrders
    }
  }
}
</script>

<style scoped>
.controls-grid {
  display: grid;
  grid-template-columns: 2fr 1fr;
  gap: 2rem;
  align-items: start;
}

.control-block {
  display: flex;
  flex-direction: column;
  gap: 0.625rem;
}

.control-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #334155;
}

.control-label .muted {
  color: #94a3b8;
  font-weight: 500;
  margin-left: 0.5rem;
}

.budget-slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 6px;
  background: #e2e8f0;
  border-radius: 999px;
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
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
}

.segmented {
  display: inline-flex;
  background: #f1f5f9;
  border-radius: 8px;
  padding: 0.25rem;
  gap: 0.25rem;
}

.seg-btn {
  padding: 0.4rem 0.875rem;
  border: none;
  background: transparent;
  border-radius: 6px;
  font-size: 0.813rem;
  font-weight: 600;
  color: #64748b;
  cursor: pointer;
  transition: all 0.15s ease;
}

.seg-btn.active {
  background: white;
  color: #0f172a;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
}

.seg-btn:hover:not(.active) {
  color: #0f172a;
}

.footer-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem 0 0;
  margin-top: 1rem;
  border-top: 1px solid #e2e8f0;
}

.totals {
  display: flex;
  gap: 1.5rem;
  font-size: 0.938rem;
  color: #334155;
}

.totals .muted {
  color: #64748b;
}

.btn-primary {
  padding: 0.625rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}

.success-banner {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem 1.25rem;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  border-radius: 10px;
  margin-bottom: 1.25rem;
}
</style>
