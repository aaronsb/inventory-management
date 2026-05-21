<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p class="subtitle">{{ t('restocking.subtitle') }}</p>
    </div>

    <div class="budget-card">
      <div class="budget-header">
        <label for="budget-slider">{{ t('restocking.budget') }}</label>
        <div class="budget-value">{{ formatCurrency(budget) }}</div>
      </div>
      <input
        id="budget-slider"
        type="range"
        min="0"
        max="50000"
        step="500"
        v-model.number="budget"
        class="budget-slider"
      />
      <div class="budget-range">
        <span>$0</span>
        <span>$50,000</span>
      </div>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="summary-row">
        <div class="summary-pill">
          <span>{{ t('restocking.items') }}</span>
          <strong>{{ recommendations.length }}</strong>
        </div>
        <div class="summary-pill">
          <span>{{ t('restocking.totalCost') }}</span>
          <strong>{{ formatCurrency(totalCost) }}</strong>
        </div>
        <div class="summary-pill">
          <span>{{ t('restocking.remaining') }}</span>
          <strong>{{ formatCurrency(Math.max(0, budget - totalCost)) }}</strong>
        </div>
        <button
          class="place-order-btn"
          :disabled="recommendations.length === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? t('common.loading') : t('restocking.placeOrder') }}
        </button>
      </div>

      <div v-if="lastSubmitted" class="confirmation">
        ✓ {{ t('restocking.submittedAs') }} <strong>{{ lastSubmitted.id }}</strong>
        — {{ t('restocking.leadTime') }}: {{ lastSubmitted.lead_time_days }} {{ t('restocking.days') }}
      </div>

      <div v-if="recommendations.length === 0" class="empty-state">
        {{ t('restocking.noRecommendations') }}
      </div>

      <div v-else class="rec-table card">
        <table>
          <thead>
            <tr>
              <th>{{ t('restocking.col.sku') }}</th>
              <th>{{ t('restocking.col.name') }}</th>
              <th>{{ t('restocking.col.category') }}</th>
              <th class="num">{{ t('restocking.col.current') }}</th>
              <th class="num">{{ t('restocking.col.forecast') }}</th>
              <th class="num">{{ t('restocking.col.shortfall') }}</th>
              <th class="num">{{ t('restocking.col.qty') }}</th>
              <th class="num">{{ t('restocking.col.unitCost') }}</th>
              <th class="num">{{ t('restocking.col.lineTotal') }}</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="r in recommendations" :key="r.sku">
              <td><code>{{ r.sku }}</code></td>
              <td>{{ r.name }}</td>
              <td>{{ r.category }}</td>
              <td class="num">{{ r.current_demand }}</td>
              <td class="num">{{ r.forecasted_demand }}</td>
              <td class="num shortfall">{{ r.shortfall }}</td>
              <td class="num qty">{{ r.recommended_qty }}</td>
              <td class="num">{{ formatCurrency(r.unit_cost) }}</td>
              <td class="num strong">{{ formatCurrency(r.line_total) }}</td>
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

export default {
  name: 'Restocking',
  setup() {
    const { t } = useI18n()
    const budget = ref(10000)
    const recommendations = ref([])
    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const lastSubmitted = ref(null)

    // Debounce the budget slider so we don't fire a request on every tick
    let debounceTimer = null

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        recommendations.value = await api.getRestockingRecommendations(budget.value)
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, r) => sum + (r.line_total || 0), 0)
    )

    const placeOrder = async () => {
      if (recommendations.value.length === 0) return
      try {
        submitting.value = true
        error.value = null
        const payload = {
          budget: budget.value,
          items: recommendations.value.map(r => ({
            sku: r.sku,
            name: r.name,
            qty: r.recommended_qty,
            unit_cost: r.unit_cost,
            line_total: r.line_total,
          }))
        }
        lastSubmitted.value = await api.submitRestockingOrder(payload)
      } catch (err) {
        error.value = 'Failed to submit order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    const formatCurrency = (value) =>
      Number(value || 0).toLocaleString('en-US', {
        style: 'currency',
        currency: 'USD',
        maximumFractionDigits: 0
      })

    watch(budget, () => {
      // Debounce: only request after the slider stops for 200ms
      clearTimeout(debounceTimer)
      debounceTimer = setTimeout(loadRecommendations, 200)
    })

    onMounted(loadRecommendations)

    return {
      t, budget, recommendations, loading, error,
      submitting, lastSubmitted, totalCost,
      placeOrder, formatCurrency,
    }
  }
}
</script>

<style scoped>
.restocking { padding: 1.5rem; }
.page-header { margin-bottom: 1.5rem; }
.page-header h2 { margin: 0 0 0.25rem; font-size: 1.5rem; color: #0f172a; }
.subtitle { margin: 0; color: #64748b; font-size: 0.875rem; }

.budget-card {
  background: #fff;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 1.25rem;
  margin-bottom: 1.5rem;
}
.budget-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.75rem;
}
.budget-header label { font-weight: 600; color: #0f172a; }
.budget-value { font-size: 1.5rem; font-weight: 700; color: #0f172a; }
.budget-slider { width: 100%; }
.budget-range {
  display: flex;
  justify-content: space-between;
  margin-top: 0.25rem;
  font-size: 0.75rem;
  color: #94a3b8;
}

.summary-row {
  display: flex;
  gap: 0.75rem;
  align-items: center;
  margin-bottom: 1rem;
  flex-wrap: wrap;
}
.summary-pill {
  background: #f1f5f9;
  border-radius: 6px;
  padding: 0.5rem 0.875rem;
  display: flex;
  flex-direction: column;
  font-size: 0.75rem;
  color: #64748b;
}
.summary-pill strong { font-size: 1.125rem; color: #0f172a; margin-top: 2px; }
.place-order-btn {
  margin-left: auto;
  background: #0f172a;
  color: #fff;
  border: 0;
  padding: 0.625rem 1.25rem;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
}
.place-order-btn:disabled { background: #cbd5e1; cursor: not-allowed; }

.confirmation {
  background: #ecfdf5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 0.625rem 0.875rem;
  border-radius: 6px;
  margin-bottom: 1rem;
  font-size: 0.875rem;
}

.empty-state {
  background: #fff;
  border: 1px dashed #cbd5e1;
  border-radius: 8px;
  padding: 2rem;
  text-align: center;
  color: #64748b;
}

.rec-table {
  background: #fff;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  overflow: hidden;
}
.rec-table table { width: 100%; border-collapse: collapse; font-size: 0.875rem; }
.rec-table th, .rec-table td {
  padding: 0.625rem 0.875rem;
  border-bottom: 1px solid #f1f5f9;
  text-align: left;
}
.rec-table th {
  background: #f8fafc;
  color: #64748b;
  font-weight: 600;
  font-size: 0.75rem;
  text-transform: uppercase;
}
.rec-table td.num, .rec-table th.num { text-align: right; }
.rec-table td.shortfall { color: #b91c1c; font-weight: 600; }
.rec-table td.qty { color: #047857; font-weight: 600; }
.rec-table td.strong { font-weight: 600; color: #0f172a; }
.rec-table code { background: #f1f5f9; padding: 2px 6px; border-radius: 3px; font-size: 0.813rem; }
.rec-table tr:last-child td { border-bottom: 0; }

.loading, .error { padding: 2rem; text-align: center; color: #64748b; }
.error { color: #b91c1c; }
</style>
