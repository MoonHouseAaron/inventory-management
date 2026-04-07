<template>
  <div class="restocking">
    <!-- Success toast -->
    <transition name="toast">
      <div v-if="showSuccess" class="success-toast">
        {{ t('restocking.orderSuccess') }}
      </div>
    </transition>

    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <!-- Budget Slider Card -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">{{ t('restocking.budget') }}</h3>
        <span class="budget-display">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
      </div>
      <div class="slider-wrapper">
        <span class="slider-label">{{ currencySymbol }}1,000</span>
        <input
          type="range"
          class="budget-slider"
          min="1000"
          max="200000"
          step="1"
          v-model.number="budget"
        />
        <span class="slider-label">{{ currencySymbol }}200,000</span>
      </div>
    </div>

    <!-- Recommendations -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">
          {{ t('restocking.recommendations') }}
          <span v-if="!loading && !error" class="item-count">({{ recommendations.length }})</span>
        </h3>
      </div>

      <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else-if="recommendations.length === 0" class="empty-state">
        {{ t('restocking.noRecommendations') }}
      </div>
      <div v-else>
        <div class="table-container">
          <table class="recommendations-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th class="col-sku">{{ t('restocking.table.sku') }}</th>
                <th class="col-item">{{ t('restocking.table.item') }}</th>
                <th class="col-category">{{ t('restocking.table.category') }}</th>
                <th class="col-trend">{{ t('restocking.table.trend') }}</th>
                <th class="col-stock">{{ t('restocking.table.stock') }} / {{ t('restocking.table.reorderPoint') }}</th>
                <th class="col-qty">{{ t('restocking.table.quantity') }}</th>
                <th class="col-unit">{{ t('restocking.table.unitCost') }}</th>
                <th class="col-total">{{ t('restocking.table.totalCost') }}</th>
                <th class="col-priority">{{ t('restocking.table.priority') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.item_sku"
                :class="{ 'row-selected': selectedSkus.has(item.item_sku) }"
              >
                <td class="col-check">
                  <label class="checkbox-label">
                    <input
                      type="checkbox"
                      class="item-checkbox"
                      :checked="selectedSkus.has(item.item_sku)"
                      @change="toggleSelection(item)"
                    />
                    <span class="checkmark"></span>
                  </label>
                </td>
                <td class="col-sku"><code class="sku-code">{{ item.item_sku }}</code></td>
                <td class="col-item">{{ translateProductName(item.item_name) }}</td>
                <td class="col-category">{{ item.category }}</td>
                <td class="col-trend">
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-stock">
                  <span :class="{ 'stock-low': item.quantity_on_hand <= item.reorder_point }">
                    {{ item.quantity_on_hand }}
                  </span>
                  <span class="stock-separator"> / </span>
                  <span class="reorder-point">{{ item.reorder_point }}</span>
                </td>
                <td class="col-qty">
                  <input
                    type="number"
                    class="qty-input"
                    :value="getQuantity(item.item_sku, item.recommended_quantity)"
                    @change="updateQuantity(item, $event)"
                    min="1"
                  />
                </td>
                <td class="col-unit">{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td class="col-total">
                  <strong>{{ currencySymbol }}{{ getLineTotal(item).toLocaleString() }}</strong>
                </td>
                <td class="col-priority">
                  <span :class="['badge', getPriorityClass(item.priority_score)]">
                    {{ getPriorityLabel(item.priority_score) }}
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Summary Bar -->
        <div class="summary-bar">
          <div class="summary-stat">
            <span class="summary-label">{{ t('restocking.summary.selectedItems') }}</span>
            <span class="summary-value">{{ selectedSkus.size }}</span>
          </div>
          <div class="summary-divider"></div>
          <div class="summary-stat">
            <span class="summary-label">{{ t('restocking.summary.totalCost') }}</span>
            <span class="summary-value cost">{{ currencySymbol }}{{ selectedTotal.toLocaleString() }}</span>
          </div>
          <div class="summary-divider"></div>
          <div class="summary-stat">
            <span class="summary-label">{{ t('restocking.summary.remainingBudget') }}</span>
            <span :class="['summary-value', remainingBudget < 0 ? 'over-budget' : 'under-budget']">
              {{ currencySymbol }}{{ remainingBudget.toLocaleString() }}
            </span>
          </div>
          <div class="summary-action">
            <button
              class="place-order-btn"
              :disabled="selectedSkus.size === 0 || submitting"
              @click="placeOrder"
            >
              <span v-if="submitting">{{ t('common.loading') }}</span>
              <span v-else>
                {{ t('restocking.placeOrder') }}
                <span v-if="selectedSkus.size > 0"> &mdash; {{ currencySymbol }}{{ selectedTotal.toLocaleString() }}</span>
              </span>
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- Submitted Orders -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">{{ t('restocking.submittedOrders') }}</h3>
      </div>

      <div v-if="loadingSubmitted" class="loading">{{ t('common.loading') }}</div>
      <div v-else-if="submittedOrders.length === 0" class="empty-state">
        {{ t('restocking.noSubmittedOrders') }}
      </div>
      <div v-else class="table-container">
        <table class="submitted-table">
          <thead>
            <tr>
              <th>{{ t('restocking.orderNumber') }}</th>
              <th>{{ t('restocking.status') }}</th>
              <th>{{ t('orders.table.items') }}</th>
              <th>{{ t('restocking.orderDate') }}</th>
              <th>{{ t('restocking.expectedDelivery') }}</th>
              <th>{{ t('restocking.leadTime') }}</th>
              <th>{{ t('restocking.totalValue') }}</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="order in submittedOrders" :key="order.id">
              <td><strong>{{ order.order_number }}</strong></td>
              <td>
                <span :class="['badge', getOrderStatusClass(order.status)]">
                  {{ t(`status.${order.status.toLowerCase()}`) || order.status }}
                </span>
              </td>
              <td>{{ t('restocking.itemsCount', { count: order.items.length }) }}</td>
              <td>{{ formatDate(order.order_date) }}</td>
              <td>{{ formatDate(order.expected_delivery) }}</td>
              <td>{{ order.lead_time_days }} {{ t('restocking.days') }}</td>
              <td><strong>{{ currencySymbol }}{{ order.total_value.toLocaleString() }}</strong></td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted, watch } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, currentLocale, translateProductName } = useI18n()

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    // Budget state
    const budget = ref(101123)
    const debounceTimer = ref(null)

    // Recommendations state
    const recommendations = ref([])
    const loading = ref(false)
    const error = ref(null)

    // Selection state: Map of sku -> quantity
    const selectedItems = ref(new Map())
    const selectedSkus = computed(() => new Set(selectedItems.value.keys()))

    // Submit state
    const submitting = ref(false)
    const showSuccess = ref(false)

    // Submitted orders state
    const submittedOrders = ref([])
    const loadingSubmitted = ref(false)

    // Computed: get quantity for a sku (falls back to recommended)
    const getQuantity = (sku, recommendedQty) => {
      if (selectedItems.value.has(sku)) {
        return selectedItems.value.get(sku).quantity
      }
      return recommendedQty
    }

    // Computed: line total respecting edited quantity
    const getLineTotal = (item) => {
      const qty = getQuantity(item.item_sku, item.recommended_quantity)
      return qty * item.unit_cost
    }

    // Computed: sum of selected items' line totals
    const selectedTotal = computed(() => {
      let total = 0
      for (const [sku, entry] of selectedItems.value.entries()) {
        total += entry.quantity * entry.unit_cost
      }
      return Math.round(total)
    })

    const remainingBudget = computed(() => budget.value - selectedTotal.value)

    // Toggle a row's selection
    const toggleSelection = (item) => {
      const map = new Map(selectedItems.value)
      if (map.has(item.item_sku)) {
        map.delete(item.item_sku)
      } else {
        map.set(item.item_sku, {
          sku: item.item_sku,
          name: item.item_name,
          quantity: item.recommended_quantity,
          unit_cost: item.unit_cost
        })
      }
      selectedItems.value = map
    }

    // Update quantity for a row (keeps selection if already selected)
    const updateQuantity = (item, event) => {
      const qty = parseInt(event.target.value, 10)
      if (isNaN(qty) || qty < 1) return
      const map = new Map(selectedItems.value)
      if (map.has(item.item_sku)) {
        map.set(item.item_sku, { ...map.get(item.item_sku), quantity: qty })
        selectedItems.value = map
      }
      // If not selected, just store the override so the input stays controlled
      // The quantity update is only meaningful when the item is selected
    }

    const getPriorityClass = (score) => {
      if (score > 2) return 'high'
      if (score >= 1) return 'medium'
      return 'low'
    }

    const getPriorityLabel = (score) => {
      if (score > 2) return t('priority.high')
      if (score >= 1) return t('priority.medium')
      return t('priority.low')
    }

    const getOrderStatusClass = (status) => {
      const map = {
        'Delivered': 'success',
        'Shipped': 'info',
        'Processing': 'warning',
        'Backordered': 'danger',
        'Pending': 'warning'
      }
      return map[status] || 'info'
    }

    const formatDate = (dateString) => {
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
      return date.toLocaleDateString(locale, {
        year: 'numeric',
        month: 'short',
        day: 'numeric'
      })
    }

    // Load recommendations from API
    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockingRecommendations(budget.value)
        recommendations.value = data
        // Clear selections that are no longer in results
        const skuSet = new Set(data.map(r => r.item_sku))
        const pruned = new Map()
        for (const [sku, entry] of selectedItems.value.entries()) {
          if (skuSet.has(sku)) pruned.set(sku, entry)
        }
        selectedItems.value = pruned
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const loadSubmittedOrders = async () => {
      loadingSubmitted.value = true
      try {
        submittedOrders.value = await api.getSubmittedOrders()
      } catch (err) {
        console.error('Failed to load submitted orders:', err)
      } finally {
        loadingSubmitted.value = false
      }
    }

    const placeOrder = async () => {
      if (selectedSkus.value.size === 0) return
      submitting.value = true
      try {
        const items = []
        for (const [sku, entry] of selectedItems.value.entries()) {
          items.push({
            item_sku: entry.sku,
            item_name: entry.name,
            quantity: entry.quantity,
            unit_cost: entry.unit_cost
          })
        }
        await api.submitRestockingOrder({
          items,
          total_budget: budget.value
        })
        // Clear selections
        selectedItems.value = new Map()
        // Show success toast
        showSuccess.value = true
        setTimeout(() => { showSuccess.value = false }, 3000)
        // Refresh submitted orders
        await loadSubmittedOrders()
      } catch (err) {
        error.value = t('restocking.orderError')
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    // Debounce budget slider changes
    watch(budget, () => {
      if (debounceTimer.value) clearTimeout(debounceTimer.value)
      debounceTimer.value = setTimeout(() => {
        loadRecommendations()
      }, 300)
    })

    onMounted(() => {
      loadRecommendations()
      loadSubmittedOrders()
    })

    return {
      t,
      currencySymbol,
      translateProductName,
      budget,
      recommendations,
      loading,
      error,
      selectedSkus,
      selectedTotal,
      remainingBudget,
      submitting,
      showSuccess,
      submittedOrders,
      loadingSubmitted,
      getQuantity,
      getLineTotal,
      toggleSelection,
      updateQuantity,
      getPriorityClass,
      getPriorityLabel,
      getOrderStatusClass,
      formatDate,
      placeOrder
    }
  }
}
</script>

<style scoped>
/* ── Success Toast ─────────────────────────────────────────── */
.success-toast {
  position: fixed;
  top: 1.5rem;
  right: 1.5rem;
  z-index: 999;
  background: #059669;
  color: #ffffff;
  padding: 0.875rem 1.5rem;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.938rem;
  box-shadow: 0 4px 12px rgba(5, 150, 105, 0.35);
}

.toast-enter-active,
.toast-leave-active {
  transition: opacity 0.3s ease, transform 0.3s ease;
}

.toast-enter-from,
.toast-leave-to {
  opacity: 0;
  transform: translateY(-0.5rem);
}

/* ── Budget Slider ─────────────────────────────────────────── */
.slider-wrapper {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0.5rem 0;
}

.slider-label {
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 500;
  white-space: nowrap;
  min-width: 60px;
}

.slider-label:last-child {
  text-align: right;
}

.budget-slider {
  flex: 1;
  -webkit-appearance: none;
  appearance: none;
  height: 6px;
  border-radius: 3px;
  background: linear-gradient(
    to right,
    #ef4444 0%,
    #ef4444 var(--progress, 24%),
    #e2e8f0 var(--progress, 24%),
    #e2e8f0 100%
  );
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #ef4444;
  border: 3px solid #ffffff;
  box-shadow: 0 0 0 1px #ef4444, 0 2px 6px rgba(239, 68, 68, 0.4);
  cursor: pointer;
  transition: box-shadow 0.2s;
}

.budget-slider::-webkit-slider-thumb:hover {
  box-shadow: 0 0 0 1px #ef4444, 0 2px 10px rgba(239, 68, 68, 0.55);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #ef4444;
  border: 3px solid #ffffff;
  box-shadow: 0 0 0 1px #ef4444, 0 2px 6px rgba(239, 68, 68, 0.4);
  cursor: pointer;
}

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

/* ── Recommendations Table ─────────────────────────────────── */
.recommendations-table {
  table-layout: fixed;
  width: 100%;
}

.col-check   { width: 40px; }
.col-sku     { width: 110px; }
.col-item    { width: 200px; }
.col-category{ width: 120px; }
.col-trend   { width: 100px; }
.col-stock   { width: 120px; }
.col-qty     { width: 100px; }
.col-unit    { width: 90px; }
.col-total   { width: 100px; }
.col-priority{ width: 90px; }

.row-selected {
  background: #eff6ff !important;
}

.sku-code {
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  font-size: 0.813rem;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  color: #475569;
}

.stock-low {
  color: #dc2626;
  font-weight: 600;
}

.stock-separator {
  color: #94a3b8;
  margin: 0 2px;
}

.reorder-point {
  color: #64748b;
}

/* ── Qty Input ─────────────────────────────────────────────── */
.qty-input {
  width: 70px;
  padding: 0.25rem 0.375rem;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.875rem;
  color: #0f172a;
  background: #ffffff;
  text-align: right;
  transition: border-color 0.15s;
}

.qty-input:focus {
  outline: none;
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.12);
}

/* ── Checkbox ──────────────────────────────────────────────── */
.checkbox-label {
  display: inline-flex;
  align-items: center;
  cursor: pointer;
  position: relative;
}

.item-checkbox {
  position: absolute;
  opacity: 0;
  width: 0;
  height: 0;
}

.checkmark {
  display: inline-block;
  width: 18px;
  height: 18px;
  border: 2px solid #cbd5e1;
  border-radius: 4px;
  background: #ffffff;
  transition: all 0.15s;
  position: relative;
}

.item-checkbox:checked + .checkmark {
  background: #2563eb;
  border-color: #2563eb;
}

.item-checkbox:checked + .checkmark::after {
  content: '';
  position: absolute;
  left: 3px;
  top: 0px;
  width: 5px;
  height: 9px;
  border: 2px solid #ffffff;
  border-top: none;
  border-left: none;
  transform: rotate(45deg);
}

.item-checkbox:focus + .checkmark {
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.2);
}

/* ── Summary Bar ───────────────────────────────────────────── */
.summary-bar {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  margin-top: 1rem;
  padding: 1rem 1.25rem;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
}

.summary-stat {
  display: flex;
  flex-direction: column;
  gap: 0.125rem;
}

.summary-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.summary-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.summary-value.cost {
  color: #2563eb;
}

.summary-value.over-budget {
  color: #dc2626;
}

.summary-value.under-budget {
  color: #059669;
}

.summary-divider {
  width: 1px;
  height: 36px;
  background: #e2e8f0;
}

.summary-action {
  margin-left: auto;
}

/* ── Place Order Button ────────────────────────────────────── */
.place-order-btn {
  padding: 0.625rem 1.5rem;
  background: #2563eb;
  color: #ffffff;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s, box-shadow 0.2s;
  white-space: nowrap;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
  box-shadow: 0 4px 12px rgba(37, 99, 235, 0.35);
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
  box-shadow: none;
}

/* ── Submitted Orders Table ────────────────────────────────── */
.submitted-table {
  width: 100%;
}

/* ── Misc ──────────────────────────────────────────────────── */
.item-count {
  font-weight: 400;
  color: #64748b;
  font-size: 0.938rem;
}

.empty-state {
  text-align: center;
  padding: 2.5rem;
  color: #94a3b8;
  font-size: 0.938rem;
}
</style>
