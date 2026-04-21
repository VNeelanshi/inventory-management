<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Manage restocking orders based on demand forecasts</p>
    </div>

    <div v-if="successMessage" class="success-banner">
      {{ successMessage }}
    </div>

    <div class="card budget-card">
      <div class="card-header">
        <h3 class="card-title">Available Budget</h3>
      </div>
      <div class="budget-controls">
        <div class="budget-slider-row">
          <span class="budget-label">$0</span>
          <input
            type="range"
            min="0"
            max="500000"
            step="1000"
            v-model.number="budget"
            class="budget-slider"
          />
          <span class="budget-label">$500K</span>
          <div class="budget-input-wrap">
            <span class="budget-input-prefix">$</span>
            <input
              type="number"
              min="0"
              max="500000"
              step="1000"
              v-model.number="budget"
              class="budget-input"
            />
          </div>
        </div>
        <div class="budget-utilization">
          <div class="utilization-track">
            <div
              class="utilization-fill"
              :style="{ width: utilizationPct + '%' }"
            ></div>
          </div>
          <span class="utilization-text">
            ${{ selectedTotalCost.toLocaleString() }} used of ${{
              budget.toLocaleString()
            }}
            &middot; {{ selectedItems.length }} items selected
          </span>
        </div>
      </div>
    </div>

    <div class="card">
      <div class="card-header">
        <h3 class="card-title">
          Recommended Items ({{ selectedItems.length }})
        </h3>
      </div>

      <div v-if="loading" class="loading">Loading...</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else>
        <div class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th>Priority</th>
                <th>SKU</th>
                <th>Item</th>
                <th>Category</th>
                <th>Trend</th>
                <th>Restock Qty</th>
                <th>Unit Cost</th>
                <th>Subtotal</th>
                <th>Lead Time</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="row in recommendationRows"
                :key="row.sku"
                :class="{ 'row-over-budget': !row.selected }"
              >
                <td>
                  <span v-if="row.selected">{{ row.priority }}</span>
                  <span v-else class="muted">—</span>
                </td>
                <td class="mono">{{ row.sku }}</td>
                <td>{{ row.name }}</td>
                <td>{{ row.category }}</td>
                <td>
                  <span :class="['badge', row.trend]">{{ row.trend }}</span>
                </td>
                <td>{{ row.quantity.toLocaleString() }}</td>
                <td>${{ row.unit_cost.toLocaleString() }}</td>
                <td>
                  <span :class="{ strikethrough: !row.selected }">
                    ${{ row.total_cost.toLocaleString() }}
                  </span>
                  <span v-if="!row.selected" class="over-budget-note"
                    >Over budget</span
                  >
                </td>
                <td>{{ row.lead_time_days }} days</td>
              </tr>
              <tr v-if="recommendationRows.length === 0">
                <td colspan="9" class="empty-state">
                  No items recommended at current budget. Increase budget or
                  check demand data.
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="action-bar">
          <span class="action-summary">
            {{ selectedItems.length }} items &middot; Total: ${{
              selectedTotalCost.toLocaleString()
            }}
          </span>
          <button
            class="btn-place-order"
            :disabled="submitting || selectedItems.length === 0"
            @click="placeOrder"
          >
            <span v-if="submitting" class="spinner"></span>
            <span v-else>Place Order</span>
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from "vue";
import { api } from "../api";

const LEAD_TIME_BY_CATEGORY = {
  "Circuit Boards": 14,
  Sensors: 10,
  Actuators: 12,
  Controllers: 10,
  "Power Supplies": 7,
  "Industrial Parts": 14,
  Mechanical: 12,
  Filters: 7,
  General: 14,
};

const SKU_PREFIX_CATEGORY = {
  PCB: "Circuit Boards",
  CTL: "Controllers",
  PSU: "Power Supplies",
  SNR: "Sensors",
  TMP: "Sensors",
  SRV: "Actuators",
  LIN: "Actuators",
  WDG: "Industrial Parts",
  BRG: "Mechanical",
  GSK: "Mechanical",
  MTR: "Actuators",
  FLT: "Filters",
  VLV: "Mechanical",
};

const SKU_PREFIX_UNIT_COST = {
  WDG: 45.0,
  BRG: 32.5,
  GSK: 8.75,
  MTR: 385.0,
  FLT: 12.5,
  VLV: 95.0,
  PSU: 78.0,
  SNR: 65.0,
  CTL: 220.0,
};

const TREND_ORDER = { increasing: 0, stable: 1, decreasing: 2 };

function getCategoryForSku(sku, invItem) {
  if (invItem) return invItem.category;
  const prefix = sku.split("-")[0];
  return SKU_PREFIX_CATEGORY[prefix] || "General";
}

function getUnitCostForSku(sku, invItem) {
  if (invItem) return invItem.unit_cost;
  const prefix = sku.split("-")[0];
  return SKU_PREFIX_UNIT_COST[prefix] || 50.0;
}

export default {
  name: "Restocking",
  setup() {
    const budget = ref(150000);
    const loading = ref(false);
    const error = ref(null);
    const submitting = ref(false);
    const successMessage = ref("");
    const successTimer = ref(null);

    const demandForecasts = ref([]);
    const inventoryItems = ref([]);

    const loadData = async () => {
      loading.value = true;
      error.value = null;
      try {
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory(),
        ]);
        demandForecasts.value = forecasts;
        inventoryItems.value = inventory;
      } catch (err) {
        error.value = "Failed to load data: " + err.message;
      } finally {
        loading.value = false;
      }
    };

    const inventoryBySku = computed(() => {
      const map = {};
      for (const item of inventoryItems.value) {
        map[item.sku] = item;
      }
      return map;
    });

    const sortedCandidates = computed(() => {
      const candidates = [];
      for (const forecast of demandForecasts.value) {
        if (forecast.forecasted_demand <= forecast.current_demand) continue;

        const inv = inventoryBySku.value[forecast.item_sku];
        const restock_qty =
          forecast.forecasted_demand - forecast.current_demand;
        const unit_cost = getUnitCostForSku(forecast.item_sku, inv);
        const category = getCategoryForSku(forecast.item_sku, inv);
        const lead_time_days = LEAD_TIME_BY_CATEGORY[category] ?? 14;

        candidates.push({
          sku: forecast.item_sku,
          name: forecast.item_name,
          category,
          trend: forecast.trend,
          quantity: restock_qty,
          unit_cost,
          total_cost: restock_qty * unit_cost,
          lead_time_days,
        });
      }

      candidates.sort((a, b) => {
        const ta = TREND_ORDER[a.trend] ?? 3;
        const tb = TREND_ORDER[b.trend] ?? 3;
        return ta - tb;
      });

      return candidates;
    });

    const recommendations = computed(() => {
      const result = [];
      let remaining = budget.value;

      for (const item of sortedCandidates.value) {
        if (item.total_cost <= remaining) {
          result.push({ ...item, selected: true });
          remaining -= item.total_cost;
        } else {
          const affordable_qty = Math.floor(remaining / item.unit_cost);
          if (affordable_qty > 0) {
            result.push({
              ...item,
              quantity: affordable_qty,
              total_cost: affordable_qty * item.unit_cost,
              selected: true,
            });
            remaining = 0;
          }
          result.push({ ...item, selected: false });
          break;
        }
      }

      return result;
    });

    const selectedItems = computed(() =>
      recommendations.value.filter((r) => r.selected),
    );

    const recommendationRows = computed(() => {
      let priority = 1;
      return recommendations.value.map((row) => ({
        ...row,
        priority: row.selected ? priority++ : null,
      }));
    });

    const selectedTotalCost = computed(() =>
      selectedItems.value.reduce((sum, item) => sum + item.total_cost, 0),
    );

    const utilizationPct = computed(() => {
      if (budget.value === 0) return 0;
      return Math.min(100, (selectedTotalCost.value / budget.value) * 100);
    });

    const placeOrder = async () => {
      if (selectedItems.value.length === 0 || submitting.value) return;
      submitting.value = true;
      try {
        const payload = {
          items: selectedItems.value.map((item) => ({
            sku: item.sku,
            name: item.name,
            category: item.category,
            quantity: item.quantity,
            unit_cost: item.unit_cost,
            total_cost: item.total_cost,
            lead_time_days: item.lead_time_days,
          })),
          total_cost: selectedTotalCost.value,
        };
        const order = await api.submitRestockingOrder(payload);

        const maxLead = Math.max(
          ...selectedItems.value.map((i) => i.lead_time_days),
        );
        successMessage.value = `Order ${order.order_number} submitted successfully! Estimated delivery in ${maxLead} days.`;

        if (successTimer.value) clearTimeout(successTimer.value);
        successTimer.value = setTimeout(() => {
          successMessage.value = "";
        }, 4000);
      } catch (err) {
        error.value = "Failed to submit order: " + err.message;
      } finally {
        submitting.value = false;
      }
    };

    onMounted(loadData);

    return {
      budget,
      loading,
      error,
      submitting,
      successMessage,
      selectedItems,
      recommendationRows,
      selectedTotalCost,
      utilizationPct,
      placeOrder,
    };
  },
};
</script>

<style scoped>
.restocking {
  padding: 0;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem;
  border-radius: 8px;
  margin-bottom: 1rem;
  font-weight: 500;
  animation: slideDown 0.3s ease;
}

@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateY(-8px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.budget-controls {
  display: flex;
  flex-direction: column;
  gap: 0.875rem;
}

.budget-slider-row {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.budget-label {
  font-size: 0.813rem;
  color: #64748b;
  white-space: nowrap;
}

.budget-slider {
  flex: 1;
  accent-color: #2563eb;
  cursor: pointer;
  height: 4px;
}

.budget-input-wrap {
  display: flex;
  align-items: center;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  background: white;
  padding: 0 0.5rem;
  min-width: 130px;
}

.budget-input-prefix {
  color: #64748b;
  font-size: 0.875rem;
}

.budget-input {
  border: none;
  outline: none;
  padding: 0.375rem 0.25rem;
  font-size: 0.875rem;
  color: #0f172a;
  width: 100%;
  font-family: inherit;
}

.budget-utilization {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.utilization-track {
  background: #e2e8f0;
  height: 6px;
  border-radius: 3px;
  overflow: hidden;
}

.utilization-fill {
  background: #2563eb;
  height: 100%;
  border-radius: 3px;
  transition: width 0.3s ease;
}

.utilization-text {
  font-size: 0.813rem;
  color: #64748b;
}

.restock-table {
  width: 100%;
  border-collapse: collapse;
  table-layout: auto;
}

.row-over-budget {
  opacity: 0.4;
}

.strikethrough {
  text-decoration: line-through;
  color: #94a3b8;
}

.over-budget-note {
  display: block;
  font-size: 0.7rem;
  color: #dc2626;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.03em;
  margin-top: 2px;
}

.mono {
  font-family: "Menlo", "Consolas", monospace;
  font-size: 0.813rem;
}

.muted {
  color: #94a3b8;
}

.empty-state {
  text-align: center;
  padding: 2rem;
  color: #64748b;
  font-size: 0.875rem;
}

.action-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 1.25rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
}

.action-summary {
  font-size: 0.875rem;
  color: #475569;
  font-weight: 500;
}

.btn-place-order {
  background: #2563eb;
  color: white;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-weight: 600;
  border: none;
  cursor: pointer;
  font-size: 0.875rem;
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  transition: background 0.15s ease;
}

.btn-place-order:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-place-order:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.spinner {
  display: inline-block;
  width: 14px;
  height: 14px;
  border: 2px solid rgba(255, 255, 255, 0.4);
  border-top-color: white;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

.budget-card {
  margin-bottom: 1.25rem;
}
</style>
