<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue'

// === 1. TYPE DEFINITIONS ===
interface SubItem {
  id: number
  name: string
  price: number
}

interface Transaction {
  id: number
  category: string        
  type: 'income' | 'expense'
  allocatedBudget: number // ទឹកប្រាក់គម្រោងដែលបានកំណត់ទុក
  date: string
  subItems: SubItem[]     // ផលិតផលជាក់ស្តែងដែលបានទិញ
}

// === 2. REACTIVE STATE ===
const transactions = ref<Transaction[]>([])
const filterType = ref<'all' | 'income' | 'expense'>('all')

// Overall Global Budget Limit
const budgetLimitInput = ref<string>('30')

// Form Inputs for master category plan
const inputCategory = ref('')
const inputType = ref<'income' | 'expense'>('expense')
const inputAllocatedBudget = ref<string>('') 

// Inline Sub-item addition state
const activeTxIdForNewSub = ref<number | null>(null)
const subItemName = ref('')
const subItemPriceInput = ref<string>('')

// Inline Editing Price State
const editingTxId = ref<number | null>(null)
const editingSubId = ref<number | null>(null)
const editPriceValue = ref<string>('0')

// === 3. COMPUTED PROPERTIES ===

const budgetLimit = computed(() => {
  const parsed = Number(budgetLimitInput.value)
  return isNaN(parsed) || parsed < 0 ? 0 : parsed
})

// Calculate Total Income
const totalIncome = computed(() => {
  return transactions.value
    .filter(t => t.type === 'income')
    .reduce((sum, t) => sum + t.subItems.reduce((s, sub) => s + sub.price, 0), 0)
})

// Calculate Total Expenses (Actual Spent)
const totalExpenses = computed(() => {
  return transactions.value
    .filter(t => t.type === 'expense')
    .reduce((sum, t) => sum + t.subItems.reduce((s, sub) => s + sub.price, 0), 0)
})

const balance = computed(() => {
  return totalIncome.value - totalExpenses.value
})

const isOverBudget = computed(() => {
  return totalExpenses.value > budgetLimit.value
})

const budgetStatus = computed(() => {
  if (isOverBudget.value) {
    return `Over spent overall by $${totalExpenses.value - budgetLimit.value}! (ចំណាយសរុបលើសលីមីតរួម $${totalExpenses.value - budgetLimit.value})`
  }
  return `Overall Budget is safe. Remaining: $${budgetLimit.value - totalExpenses.value}`
})

// Generate color blocks based on Actual Spending + Map target and budget surplus/deficit status
const progressBarBlocks = computed(() => {
  if (budgetLimit.value <= 0) return []
  const colors = ['#2ecc71', '#f39c12', '#e74c3c', '#9b59b6', '#1abc9c', '#34495e']
  let colorIndex = 0
  
  return transactions.value
    .filter(t => t.type === 'expense')
    .map(t => {
      const actualAmount = t.subItems.reduce((sum, sub) => sum + sub.price, 0)
      const pct = (actualAmount / budgetLimit.value) * 100
      const blockColor = colors[colorIndex % colors.length]
      colorIndex++
      
      // គណនាចំនួនទឹកប្រាក់ដែលលើស ឬនៅសល់សម្រាប់ Category នីមួយៗ
      const difference = actualAmount - t.allocatedBudget
      let statusText = ''
      if (difference > 0) {
        statusText = `Over Plan by +$${difference} 🔴`
      } else if (difference < 0) {
        statusText = `Remaining: $${Math.abs(difference)} 🟢`
      } else {
        statusText = 'On Target 🟢'
      }
      
      return {
        category: t.category,
        plannedBudget: t.allocatedBudget, 
        actualAmount: actualAmount,       
        percentage: pct,
        color: blockColor,
        statusText: statusText // បញ្ជូនអត្ថបទស្ថានភាពទៅកាន់ Tooltip
      }
    })
    .filter(block => block.actualAmount > 0) 
})

const totalExpensePercentage = computed(() => {
  if (budgetLimit.value <= 0) return 0
  return (totalExpenses.value / budgetLimit.value) * 100
})

const filteredTransactions = computed(() => {
  if (filterType.value === 'all') return transactions.value
  return transactions.value.filter(t => t.type === filterType.value)
})

// === 4. HELPER FUNCTIONS ===
const getActualTxTotal = (tx: Transaction) => {
  return tx.subItems.reduce((sum, sub) => sum + sub.price, 0)
}

// === 5. METHODS ===
const addTransaction = () => {
  const budgetParsed = Number(inputAllocatedBudget.value)
  if (!inputCategory.value.trim() || isNaN(budgetParsed) || budgetParsed < 0) {
    alert('Please enter a category name and a valid planned target budget!')
    return
  }

  transactions.value.push({
    id: Date.now(),
    category: inputCategory.value.trim().toLowerCase(),
    type: inputType.value,
    allocatedBudget: budgetParsed,
    date: new Date().toLocaleDateString(),
    subItems: [] 
  })

  inputCategory.value = ''
  inputAllocatedBudget.value = ''
}

const addSubItemToTransaction = (txId: number) => {
  const priceParsed = Number(subItemPriceInput.value)
  if (!subItemName.value.trim() || isNaN(priceParsed) || priceParsed <= 0) {
    alert('Please enter product name and positive price!')
    return
  }

  const targetTx = transactions.value.find(t => t.id === txId)
  if (targetTx) {
    targetTx.subItems.push({
      id: Date.now() + Math.random(),
      name: subItemName.value.trim(),
      price: priceParsed
    })
  }

  subItemName.value = ''
  subItemPriceInput.value = ''
  activeTxIdForNewSub.value = null
}

const deleteSubItem = (txId: number, subId: number) => {
  const targetTx = transactions.value.find(t => t.id === txId)
  if (targetTx) {
    targetTx.subItems = targetTx.subItems.filter(s => s.id !== subId)
  }
}

const deleteTransactionGroup = (id: number) => {
  transactions.value = transactions.value.filter(t => t.id !== id)
}

const startEditPrice = (txId: number, subId: number, currentPrice: number) => {
  editingTxId.value = txId
  editingSubId.value = subId
  editPriceValue.value = String(currentPrice)
}

const savePriceEdit = (txId: number, subId: number) => {
  const priceParsed = Number(editPriceValue.value)
  if (isNaN(priceParsed) || priceParsed < 0) {
    alert('Please enter a valid amount!')
    return
  }

  const tx = transactions.value.find(t => t.id === txId)
  if (tx) {
    const sub = tx.subItems.find(s => s.id === subId)
    if (sub) {
      sub.price = priceParsed
    }
  }
  cancelEdit()
}

const cancelEdit = () => {
  editingTxId.value = null
  editingSubId.value = null
}

const clearAll = () => {
  if (confirm('Wipe all tracking histories?')) {
    transactions.value = []
  }
}

// === 6. WATCHERS & LIFECYCLE ===
watch(transactions, (newVal) => {
  localStorage.setItem('finance_v6_data', JSON.stringify(newVal))
}, { deep: true })

watch(budgetLimitInput, (newVal) => {
  localStorage.setItem('finance_v6_budget', newVal)
})

onMounted(() => {
  const savedData = localStorage.getItem('finance_v6_data')
  if (savedData) {
    transactions.value = JSON.parse(savedData)
  }
  const savedBudget = localStorage.getItem('finance_v6_budget')
  if (savedBudget !== null) {
    budgetLimitInput.value = savedBudget
  }
})
</script>

<template>
  <div class="app-container">
    <h1 class="main-title">Personal Finance Tracker</h1>

    <div class="section-card configuration">
      <div class="input-inline">
        <label>Set Global Budget Limit ($): </label>
        <input 
          type="text" 
          inputmode="numeric"
          v-model="budgetLimitInput" 
          class="input-style primary-input" 
          placeholder="Type global limit..."
        />
      </div>
      <div class="status-box" :class="{ 'alert-danger': isOverBudget, 'alert-success': !isOverBudget }">
        <strong>Target Analysis: </strong> {{ budgetStatus }}
      </div>
    </div>

    <div class="section-card visual-chart">
      <h3>Multi-Color Expense Allocation Bar (Actual Spent)</h3>
      <p class="instruction-text">*Hover mouse over color blocks to view Planned Target, Actual Spent, and Over/Remaining balance</p>
      
      <div class="segmented-progress-bg">
        <div 
          v-for="block in progressBarBlocks" 
          :key="block.category" 
          class="progress-segment"
          :style="{ width: block.percentage + '%', backgroundColor: block.color }"
          :title="`Category: ${block.category.toUpperCase()}\nPlanned Target: $${block.plannedBudget}\nActual Spent: $${block.actualAmount} (${block.percentage.toFixed(1)}%)\nStatus: ${block.statusText}`"
        ></div>
      </div>

      <div class="color-legends">
        <span 
          v-for="block in progressBarBlocks" 
          :key="block.category" 
          class="legend-item" 
          :title="`Planned Target: $${block.plannedBudget} | Status: ${block.statusText}`"
        >
          <span class="color-dot" :style="{ backgroundColor: block.color }"></span>
          {{ block.category }}: ${{ block.actualAmount }} ({{ block.percentage.toFixed(1) }})
        </span>
        <span v-if="progressBarBlocks.length === 0" class="empty-text">No active actual spending recorded yet</span>
      </div>
      <p class="total-text">Global Budget Used: <strong>{{ totalExpensePercentage.toFixed(1) }}%</strong></p>
    </div>

    <div class="grid-summary">
      <div class="summary-item total-income">Total Income: +${{ totalIncome }}</div>
      <div class="summary-item total-expense">Total Expense: -${{ totalExpenses }}</div>
      <div class="summary-item net-balance" :class="{ 'minus-value': balance < 0 }">Actual Balance: ${{ balance }}</div>
    </div>

    <div class="section-card input-form">
      <h3>Set Pre-planned Category Target Budget</h3>
      <div class="form-row-flexible">
        <input 
          type="text" 
          v-model="inputCategory" 
          placeholder="Category name (e.g., skincare, clothes)" 
          class="input-style" 
        />
        <input 
          type="text" 
          inputmode="numeric"
          v-model="inputAllocatedBudget" 
          placeholder="Target budget ($)" 
          class="input-style budget-init-box" 
        />
        <select v-model="inputType" class="input-style select-box">
          <option value="income">Income</option>
          <option value="expense">Expense</option>
        </select>
        <button @click="addTransaction" class="btn-primary-action">Set Plan</button>
      </div>
    </div>

    <div class="filter-controls">
      <label>Filter Records: </label>
      <button @click="filterType = 'all'" :class="{ active: filterType === 'all' }">All</button>
      <button @click="filterType = 'income'" :class="{ active: filterType === 'income' }">Income Only</button>
      <button @click="filterType = 'expense'" :class="{ active: filterType === 'expense' }">Expense Only</button>
      <button @click="clearAll" class="btn-danger-clear">Clear All Histories</button>
    </div>

    <div class="section-card history-list">
      <h3>Transaction Tracking Matrix Logs</h3>
      
      <div v-for="item in filteredTransactions" :key="item.id" class="category-block">
        <div class="category-header">
          <span class="cat-title">
            <strong>[{{ item.type.toUpperCase() }}]</strong> 
            <span class="highlight-cat-text">{{ item.category }}</span> 
            <small class="date-stamp">({{ item.date }})</small>
          </span>
          
          <div class="header-action-buttons">
            <button @click="activeTxIdForNewSub = item.id" class="btn-add-product-inline">+ Add Product Spend</button>
            <button @click="deleteTransactionGroup(item.id)" class="btn-mini-delete">Delete Group</button>
          </div>
        </div>

        <div class="budget-comparison-bar" v-if="item.type === 'expense'">
          <span>Planned Target: <strong>${{ item.allocatedBudget }}</strong></span>
          <span>Actual Spent: <strong :class="item.type">${{ getActualTxTotal(item) }}</strong></span>
          
          <span v-if="getActualTxTotal(item) > item.allocatedBudget" class="badge-status over-spending">
            Over Plan by +${{ getActualTxTotal(item) - item.allocatedBudget }} 
          </span>
          <span v-else-if="getActualTxTotal(item) < item.allocatedBudget" class="badge-status safe-spending">
            Remaining: ${{- (getActualTxTotal(item) - item.allocatedBudget) }} 
          </span>
          <span v-else class="badge-status exact-spending">
            On Target 
          </span>
        </div>

        <div v-if="activeTxIdForNewSub === item.id" class="inline-subform-box">
          <h5>Purchase product entry for "{{ item.category }}"</h5>
          <div class="form-row-flexible">
            <input type="text" v-model="subItemName" placeholder="Product item (e.g., nightcream, sun screen)" class="input-style text-mini" />
            <input type="text" inputmode="numeric" v-model="subItemPriceInput" placeholder="Price ($)" class="input-style text-mini" />
            <button @click="addSubItemToTransaction(item.id)" class="btn-save">Confirm Add</button>
            <button @click="activeTxIdForNewSub = null" class="btn-cancel">Cancel</button>
          </div>
        </div>
        
        <table class="sub-products-table">
          <thead>
            <tr>
              <th>Product/Item Item Description</th>
              <th>Price Amount</th>
              <th class="text-right">Actions</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="sub in item.subItems" :key="sub.id">
              <td class="product-name-cell">- {{ sub.name }}</td>
              <td>
                <div v-if="editingTxId === item.id && editingSubId === sub.id">
                  <input type="text" inputmode="numeric" v-model="editPriceValue" class="input-mini-edit" />
                </div>
                <div v-else :class="item.type" class="price-display">
                  ${{ sub.price }}
                </div>
              </td>
              <td class="text-right">
                <div v-if="editingTxId === item.id && editingSubId === sub.id">
                  <button @click="savePriceEdit(item.id, sub.id)" class="btn-save-mini">Save</button>
                  <button @click="cancelEdit" class="btn-cancel-mini">Cancel</button>
                </div>
                <div v-else>
                  <button @click="startEditPrice(item.id, sub.id, sub.price)" class="btn-edit-mini">Edit Price</button>
                  <button @click="deleteSubItem(item.id, sub.id)" class="btn-remove-mini">Remove</button>
                </div>
              </td>
            </tr>
            <tr v-if="item.subItems.length === 0">
              <td colspan="3" class="no-products-placeholder">No actual products bought yet. Click "+ Add Product Spend" to enter nested values.</td>
            </tr>
          </tbody>
        </table>
      </div>
      
      <div v-if="filteredTransactions.length === 0" class="empty-text">No customized categories matched.</div>
    </div>
  </div>
</template>

<style scoped>
.app-container { max-width: 780px; margin: 20px auto; padding: 20px; font-family: 'Segoe UI', system-ui, sans-serif; color: #333; background-color: #fafafa; }
.main-title { text-align: center; color: #2c3e50; font-size: 26px; margin-bottom: 25px; font-weight: 700; }
.section-card { background: #ffffff; padding: 20px; border-radius: 10px; box-shadow: 0 4px 6px rgba(0,0,0,0.05); margin-bottom: 20px; border: 1px solid #eaeaea; }
.input-inline { display: flex; align-items: center; gap: 12px; margin-bottom: 12px; }
.input-style { padding: 10px 14px; font-size: 14px; border: 1px solid #cbd5e1; border-radius: 6px; flex: 1; outline: none; background: #fff; }
.input-style:focus { border-color: #3498db; box-shadow: 0 0 0 2px rgba(52,152,219,0.2); }
.primary-input { font-weight: bold; font-size: 15px; color: #2c3e50; }
.budget-init-box { max-width: 150px; }
.instruction-text { font-size: 11px; color: #7f8c8d; margin: -5px 0 5px 0; font-style: italic; }
.status-box { padding: 12px 16px; border-radius: 6px; font-size: 14px; margin-top: 10px; line-height: 1.5; }
.alert-danger { background-color: #fce8e6; color: #c5221f; border: 1px solid #fad2cf; }
.alert-success { background-color: #e6f4ea; color: #137333; border: 1px solid #ceead6; }

/* Progress Multi-color layouts */
.segmented-progress-bg { background-color: #eef2f3; width: 100%; height: 28px; border-radius: 14px; display: flex; overflow: hidden; margin: 15px 0; box-shadow: inset 0 1px 3px rgba(0,0,0,0.1); cursor: help; }
.progress-segment { height: 100%; transition: width 0.4s ease; position: relative; }
.color-legends { display: flex; flex-wrap: wrap; gap: 12px; font-size: 13px; margin-top: 10px; }
.legend-item { display: flex; align-items: center; gap: 6px; background: #f1f3f4; padding: 4px 10px; border-radius: 4px; font-weight: 500; cursor: help; }
.color-dot { width: 12px; height: 12px; border-radius: 50%; display: inline-block; }
.total-text { margin-top: 12px; font-size: 13px; text-align: right; color: #666; }

.grid-summary { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 15px; margin-bottom: 20px; }
.summary-item { padding: 15px; border-radius: 8px; text-align: center; font-weight: bold; font-size: 15px; color: white; box-shadow: 0 2px 4px rgba(0,0,0,0.05); }
.total-income { background-color: #2ecc71; }
.total-expense { background-color: #e74c3c; }
.net-balance { background-color: #3498db; }
.net-balance.minus-value { background-color: #962d22; }

.form-row-flexible { display: flex; gap: 10px; align-items: center; width: 100%; }
.select-box { max-width: 110px; cursor: pointer; }
.btn-primary-action { background: #2c3e50; color: white; border: none; padding: 11px 18px; font-weight: bold; border-radius: 6px; cursor: pointer; font-size: 13px; white-space: nowrap; }
.btn-primary-action:hover { background: #1a252f; }

.filter-controls { display: flex; gap: 8px; align-items: center; margin-bottom: 20px; font-size: 14px; }
.filter-controls button { padding: 6px 14px; border: 1px solid #cbd5e1; background: white; border-radius: 6px; cursor: pointer; font-weight: 500; }
.filter-controls button.active { background: #007bff; color: white; border-color: #007bff; }
.btn-danger-clear { background: #475569; color: white; border: none; padding: 7px 14px; border-radius: 6px; cursor: pointer; margin-left: auto; font-size: 12px; }

.category-block { background: #ffffff; border: 1px solid #e2e8f0; border-radius: 8px; padding: 16px; margin-bottom: 15px; box-shadow: 0 2px 4px rgba(0,0,0,0.02); }
.category-header { display: flex; justify-content: space-between; align-items: center; border-bottom: 2px solid #f1f5f9; padding-bottom: 10px; margin-bottom: 8px; }
.highlight-cat-text { text-transform: capitalize; font-size: 16px; font-weight: 600; color: #1e293b; }
.date-stamp { color: #94a3b8; margin-left: 4px; }
.header-action-buttons { display: flex; gap: 8px; }

.budget-comparison-bar { display: flex; gap: 15px; background: #f8fafc; padding: 8px 12px; border-radius: 6px; font-size: 13px; margin-bottom: 12px; align-items: center; border: 1px solid #f1f5f9; }
.badge-status { padding: 2px 8px; border-radius: 4px; font-weight: 600; font-size: 11px; margin-left: auto; }
.over-spending { background: #fee2e2; color: #ef4444; }
.safe-spending { background: #e6f4ea; color: #137333; }
.exact-spending { background: #fef3c7; color: #d97706; }

.btn-add-product-inline { background: #3498db; color: white; border: none; padding: 4px 10px; border-radius: 4px; cursor: pointer; font-size: 12px; font-weight: 600; }
.btn-mini-delete { background: #fee2e2; color: #ef4444; border: 1px solid #fca5a5; padding: 4px 10px; border-radius: 4px; cursor: pointer; font-size: 12px; }

.inline-subform-box { background: #f8fafc; border: 1px dashed #3498db; border-radius: 6px; padding: 12px; margin-bottom: 12px; }
.inline-subform-box h5 { margin: 0 0 8px 0; font-size: 13px; color: #34495e; }
.text-mini { padding: 6px 10px; font-size: 12px; }

.sub-products-table { width: 100%; border-collapse: collapse; font-size: 13px; }
.sub-products-table th, .sub-products-table td { padding: 8px; text-align: left; border-bottom: 1px solid #f1f5f9; }
.sub-products-table th { color: #64748b; font-weight: 600; background: #f8fafc; font-size: 12px; }
.product-name-cell { color: #334155; padding-left: 10px; }
.price-display { font-weight: 600; }
.income { color: #2ecc71; }
.expense { color: #e74c3c; }

.text-right { text-align: right !important; }
.input-mini-edit { width: 80px; padding: 4px 6px; font-size: 12px; border: 1px solid #3498db; border-radius: 4px; }
.btn-edit-mini { background: #fff; color: #d35400; border: 1px solid #f39c12; padding: 3px 8px; border-radius: 4px; cursor: pointer; font-size: 11px; margin-right: 4px; }
.btn-remove-mini { background: #fff; color: #7f8c8d; border: 1px solid #bdc3c7; padding: 3px 8px; border-radius: 4px; cursor: pointer; font-size: 11px; }
.btn-save-mini { background: #2ecc71; color: white; border: none; padding: 3px 8px; border-radius: 4px; cursor: pointer; font-size: 11px; margin-right: 4px; }
.btn-cancel-mini { background: #95a5a6; color: white; border: none; padding: 3px 8px; border-radius: 4px; cursor: pointer; font-size: 11px; }

.no-products-placeholder { text-align: center; color: #94a3b8; font-style: italic; padding: 15px 0; }
.empty-text { font-style: italic; color: #94a3b8; text-align: center; font-size: 14px; padding: 10px; }
</style>