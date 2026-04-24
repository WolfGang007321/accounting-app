<script setup>
import { ref, computed, onMounted, watch, nextTick } from 'vue'
import { Chart, ArcElement, Tooltip, Legend, DoughnutController } from 'chart.js'

Chart.register(ArcElement, Tooltip, Legend, DoughnutController)

// 收支记录列表
const records = ref([])

// 当前选中月份（默认本月）
const now = new Date()
const selectedYear = ref(now.getFullYear())
const selectedMonth = ref(now.getMonth())

// 预算
const monthlyBudget = ref(0)

// 编辑状态
const editId = ref(null)

// 表单数据
const form = ref({
  type: 'expense',
  amount: '',
  category: '餐饮',
  date: new Date().toISOString().split('T')[0],
  note: ''
})

// 分类选项（带 emoji）
const expenseCategories = ['餐饮', '交通', '购物', '娱乐', '居住', '医疗', '其他']
const incomeCategories = ['工资', '兼职', '理财', '红包', '其他']
const categoryIcons = {
  '餐饮': '🍜', '交通': '🚌', '购物': '🛒', '娱乐': '🎮',
  '居住': '🏠', '医疗': '💊', '其他': '📦',
  '工资': '💰', '兼职': '💼', '理财': '📈', '红包': '🧧'
}

const categories = computed(() => form.value.type === 'expense' ? expenseCategories : incomeCategories)

// 提交表单（添加或编辑）
function submitForm() {
  if (!form.value.amount || form.value.amount <= 0) {
    alert('请输入正确的金额')
    return
  }
  if (editId.value !== null) {
    // 编辑模式：更新记录
    const idx = records.value.findIndex(r => r.id === editId.value)
    if (idx !== -1) {
      records.value[idx] = {
        ...records.value[idx],
        type: form.value.type,
        amount: Number(form.value.amount),
        category: form.value.category,
        date: form.value.date,
        note: form.value.note
      }
    }
    editId.value = null
  } else {
    // 添加模式
    records.value.unshift({
      id: Date.now(),
      type: form.value.type,
      amount: Number(form.value.amount),
      category: form.value.category,
      date: form.value.date,
      note: form.value.note
    })
  }
  saveData()
  resetForm()
}

// 编辑记录
function editRecord(record) {
  editId.value = record.id
  form.value.type = record.type
  form.value.amount = String(record.amount)
  form.value.category = record.category
  form.value.date = record.date
  form.value.note = record.note || ''
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

// 删除记录
function deleteRecord(id) {
  if (!confirm('确定要删除这条记录吗？')) return
  records.value = records.value.filter(r => r.id !== id)
  saveData()
}

// 取消编辑
function cancelEdit() {
  editId.value = null
  resetForm()
}

// 重置表单
function resetForm() {
  form.value.amount = ''
  form.value.note = ''
  form.value.date = new Date().toISOString().split('T')[0]
}

// 统计
const totalIncome = computed(() =>
  records.value.filter(r => r.type === 'income').reduce((sum, r) => sum + r.amount, 0)
)
const totalExpense = computed(() =>
  records.value.filter(r => r.type === 'expense').reduce((sum, r) => sum + r.amount, 0)
)
const balance = computed(() => totalIncome.value - totalExpense.value)

// 选中月份的记录
const selectedMonthRecords = computed(() =>
  records.value.filter(r => {
    const d = new Date(r.date)
    return d.getMonth() === selectedMonth.value && d.getFullYear() === selectedYear.value
  })
)
const monthIncome = computed(() =>
  selectedMonthRecords.value.filter(r => r.type === 'income').reduce((sum, r) => sum + r.amount, 0)
)
const monthExpense = computed(() =>
  selectedMonthRecords.value.filter(r => r.type === 'expense').reduce((sum, r) => sum + r.amount, 0)
)
const monthBalance = computed(() => monthIncome.value - monthExpense.value)

// 预算进度
const budgetUsed = computed(() =>
  monthlyBudget.value > 0
    ? Math.min((monthExpense.value / monthlyBudget.value) * 100, 100)
    : 0
)
const isOverBudget = computed(() =>
  monthlyBudget.value > 0 && monthExpense.value > monthlyBudget.value
)
const budgetRemaining = computed(() =>
  monthlyBudget.value > 0 ? monthlyBudget.value - monthExpense.value : 0
)

// 本月分类统计
const monthByCategory = computed(() => {
  const result = {}
  selectedMonthRecords.value.forEach(r => {
    if (!result[r.category]) result[r.category] = 0
    result[r.category] += r.amount
  })
  return Object.entries(result).sort((a, b) => b[1] - a[1])
})

// 选中月的天数和当前天
const monthDays = computed(() =>
  new Date(selectedYear.value, selectedMonth.value + 1, 0).getDate()
)
const currentDay = computed(() => {
  const today = new Date()
  return (today.getFullYear() === selectedYear.value && today.getMonth() === selectedMonth.value)
    ? today.getDate() : monthDays.value
})
const avgDailyExpense = computed(() =>
  currentDay.value > 0 ? (monthExpense.value / currentDay.value).toFixed(1) : '0.0'
)

// 月份切换
function prevMonth() {
  if (selectedMonth.value === 0) {
    selectedMonth.value = 11
    selectedYear.value--
  } else {
    selectedMonth.value--
  }
}
function nextMonth() {
  const now2 = new Date()
  const isCurrentMonth = selectedYear.value === now2.getFullYear() && selectedMonth.value === now2.getMonth()
  if (isCurrentMonth) return
  if (selectedMonth.value === 11) {
    selectedMonth.value = 0
    selectedYear.value++
  } else {
    selectedMonth.value++
  }
}
const isCurrentMonth = computed(() => {
  const now2 = new Date()
  return selectedYear.value === now2.getFullYear() && selectedMonth.value === now2.getMonth()
})
const selectedMonthLabel = computed(() => {
  return new Date(selectedYear.value, selectedMonth.value, 1).toLocaleDateString('zh-CN', { year: 'numeric', month: 'long' })
})

// 保存到 localStorage
function saveData() {
  localStorage.setItem('accounting_records', JSON.stringify(records.value))
  localStorage.setItem('accounting_budget', String(monthlyBudget.value))
}

// 读取 localStorage
function loadData() {
  const saved = localStorage.getItem('accounting_records')
  if (saved) {
    records.value = JSON.parse(saved)
  }
  const budget = localStorage.getItem('accounting_budget')
  if (budget) {
    monthlyBudget.value = Number(budget)
  }
}

// 格式化金额
function formatMoney(n) {
  return n.toLocaleString('zh-CN', { minimumFractionDigits: 2, maximumFractionDigits: 2 })
}

// 切换类型时更新分类
function onTypeChange() {
  form.value.category = categories.value[0]
}

loadData()

// 饼图
const chartCanvas = ref(null)
let chartInstance = null

const chartColors = [
  '#ff6384', '#36a2eb', '#ffce56', '#4bc0c0', '#9966ff',
  '#ff9f40', '#ff6384', '#c9cbcf', '#7cb305', '#00a8cc'
]

function initChart() {
  if (!chartCanvas.value) return
  const ctx = chartCanvas.value.getContext('2d')
  const data = monthByCategory.value
  chartInstance = new Chart(ctx, {
    type: 'doughnut',
    data: {
      labels: data.map(([cat]) => cat),
      datasets: [{
        data: data.map(([, amount]) => amount),
        backgroundColor: chartColors.slice(0, data.length),
        borderWidth: 0
      }]
    },
    options: {
      responsive: true,
      maintainAspectRatio: true,
      cutout: '60%',
      plugins: {
        legend: { display: false },
        tooltip: {
          callbacks: {
            label: (ctx) => ` ${ctx.label}: ¥${ctx.raw.toLocaleString()}`
          }
        }
      }
    }
  })
}

function updateChart() {
  if (!chartInstance) return
  const data = monthByCategory.value
  chartInstance.data.labels = data.map(([cat]) => cat)
  chartInstance.data.datasets[0].data = data.map(([, amount]) => amount)
  chartInstance.data.datasets[0].backgroundColor = chartColors.slice(0, data.length)
  chartInstance.update()
}

onMounted(() => {
  if (monthByCategory.value.length > 0) {
    nextTick(initChart)
  }
})

watch(monthByCategory, (newVal) => {
  if (newVal.length > 0) {
    nextTick(() => {
      if (chartInstance) updateChart()
      else initChart()
    })
  }
}, { deep: true })
</script>

<template>
  <div class="app">
    <!-- 头部 + 月份切换 -->
    <header class="header">
      <div class="total-balance">
        <span class="total-label">💎 累计结余</span>
        <span class="total-amount" :class="{ negative: balance < 0 }">
          {{ balance >= 0 ? '+' : '' }}¥{{ formatMoney(balance) }}
        </span>
      </div>
      <h1>记账本</h1>
      <div class="month-nav">
        <button @click="prevMonth">‹</button>
        <p class="date">{{ selectedMonthLabel }}</p>
        <button @click="nextMonth" :disabled="isCurrentMonth">›</button>
      </div>
    </header>

    <!-- 本月概览 -->
    <section class="summary">
      <div class="summary-card income">
        <span>收入</span>
        <strong>+{{ formatMoney(monthIncome) }}</strong>
      </div>
      <div class="summary-card expense">
        <span>支出</span>
        <strong :class="{ 'over-budget': isOverBudget }">-{{ formatMoney(monthExpense) }}</strong>
      </div>
      <div class="summary-card balance" :class="{ positive: monthBalance >= 0, negative: monthBalance < 0 }">
        <span>结余</span>
        <strong>{{ formatMoney(monthBalance) }}</strong>
      </div>
    </section>

    <!-- 预算进度条 -->
    <section class="budget-section" v-if="monthlyBudget > 0">
      <div class="budget-header">
        <span>💰 月预算</span>
        <div class="budget-input-wrap">
          <span>¥</span>
          <input v-model.number="monthlyBudget" type="number" step="100" class="budget-input" @change="saveData" placeholder="设置预算" />
        </div>
      </div>
      <div class="budget-bar">
        <div
          class="budget-fill"
          :class="{ 'over': isOverBudget }"
          :style="{ width: budgetUsed + '%' }"
        ></div>
      </div>
      <div class="budget-info">
        <span v-if="!isOverBudget">还剩 ¥{{ formatMoney(budgetRemaining) }} 可花</span>
        <span v-else class="over-text">⚠️ 已超支 ¥{{ formatMoney(-budgetRemaining) }}</span>
      </div>
    </section>

    <!-- 每日平均支出 -->
    <section class="avg-section">
      <div class="avg-item">
        <span class="label">日均支出</span>
        <span class="value">¥{{ avgDailyExpense }}</span>
      </div>
      <div class="avg-item">
        <span class="label">剩余天数</span>
        <span class="value">{{ monthDays - currentDay }}天</span>
      </div>
    </section>

    <!-- 添加/编辑记录 -->
    <section class="form-section">
      <h2>{{ editId !== null ? '✏️ 编辑记录' : '记一笔' }}</h2>
      <div class="type-toggle">
        <button :class="{ active: form.type === 'expense' }" @click="form.type = 'expense'; onTypeChange()">支出</button>
        <button :class="{ active: form.type === 'income' }" @click="form.type = 'income'; onTypeChange()">收入</button>
      </div>
      <form @submit.prevent="submitForm" class="record-form">
        <div class="form-row">
          <input
            v-model="form.amount"
            type="number"
            step="0.01"
            placeholder="金额"
            class="amount-input"
          />
          <select v-model="form.category">
            <option v-for="cat in categories" :key="cat" :value="cat">{{ cat }}</option>
          </select>
        </div>
        <div class="form-row">
          <input v-model="form.date" type="date" />
          <input v-model="form.note" type="text" placeholder="备注（选填）" />
        </div>
        <div class="form-actions">
          <button v-if="editId !== null" type="button" class="cancel-btn" @click="cancelEdit">取消</button>
          <button type="submit" class="submit-btn" :class="form.type">
            {{ editId !== null ? '💾 保存' : (form.type === 'expense' ? '- 记支出' : '+ 记收入') }}
          </button>
        </div>
      </form>
    </section>

    <!-- 分类统计 -->
    <section class="category-section" v-if="monthByCategory.length">
      <h2>本月支出分布</h2>
      <div class="chart-wrapper">
        <canvas ref="chartCanvas"></canvas>
      </div>
      <div class="category-list">
        <div v-for="[cat, amount] in monthByCategory" :key="cat" class="category-item">
          <div class="cat-left">
            <span class="cat-icon">{{ categoryIcons[cat] || '📦' }}</span>
            <span>{{ cat }}</span>
          </div>
          <span class="cat-amount">¥{{ formatMoney(amount) }}</span>
        </div>
      </div>
    </section>

    <!-- 记录列表 -->
    <section class="records-section">
      <h2>本月记录</h2>
      <div v-if="selectedMonthRecords.length === 0" class="empty">
        本月还没有记录
      </div>
      <div v-else class="record-list">
        <div v-for="record in selectedMonthRecords" :key="record.id" class="record-item" :class="record.type">
          <div class="record-left">
            <span class="record-icon">{{ categoryIcons[record.category] || '📦' }}</span>
            <div class="record-left-text">
              <span class="record-cat">{{ record.category }}</span>
              <span class="record-note">{{ record.note || record.date }}</span>
            </div>
          </div>
          <div class="record-right">
            <span class="record-amount" :class="record.type">
              {{ record.type === 'income' ? '+' : '-' }}{{ formatMoney(record.amount) }}
            </span>
            <button class="edit-btn" @click="editRecord(record)">✏️</button>
            <button class="delete-btn" @click="deleteRecord(record.id)">×</button>
          </div>
        </div>
      </div>
    </section>
  </div>
</template>

<style>
* { box-sizing: border-box; margin: 0; padding: 0; }

body {
  font-family: -apple-system, BlinkMacSystemFont, 'PingFang SC', 'Microsoft YaHei', sans-serif;
  background: #f0f2f5;
  color: #1a1a2e;
  -webkit-font-smoothing: antialiased;
}

.app {
  max-width: 480px;
  margin: 0 auto;
  padding: 0 0 80px;
}

/* 渐变头部 */
.header {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 20px 20px 24px;
  border-radius: 0 0 24px 24px;
  text-align: center;
  margin-bottom: 20px;
  box-shadow: 0 4px 20px rgba(102, 126, 234, 0.35);
}
.total-balance {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-bottom: 8px;
}
.total-label { font-size: 12px; opacity: 0.75; font-weight: 500; letter-spacing: 0.5px; text-transform: uppercase; }
.total-amount { font-size: 28px; font-weight: 700; letter-spacing: -1px; }
.total-amount.negative { color: #fca5a5; }
.header h1 {
  font-size: 22px;
  font-weight: 700;
  letter-spacing: -0.5px;
  margin-bottom: 16px;
}
.month-nav {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 16px;
}
.month-nav p.date {
  font-size: 15px;
  font-weight: 500;
  opacity: 0.9;
  min-width: 110px;
}
.month-nav button {
  background: rgba(255,255,255,0.2);
  border: none;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  font-size: 20px;
  cursor: pointer;
  color: white;
  line-height: 1;
  transition: background 0.2s;
  backdrop-filter: blur(4px);
}
.month-nav button:hover { background: rgba(255,255,255,0.3); }
.month-nav button:disabled { opacity: 0.3; cursor: default; }

/* 概览卡片 */
.summary {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 10px;
  margin: 0 16px 12px;
}
.summary-card {
  background: white;
  border-radius: 16px;
  padding: 14px 10px;
  text-align: center;
  box-shadow: 0 2px 8px rgba(0,0,0,0.06);
  transition: transform 0.2s, box-shadow 0.2s;
}
.summary-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 16px rgba(0,0,0,0.1);
}
.summary-card span {
  font-size: 11px;
  color: #9ca3af;
  display: block;
  margin-bottom: 6px;
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}
.summary-card strong { font-size: 15px; font-weight: 700; }
.summary-card.income strong { color: #22c55e; }
.summary-card.expense strong { color: #ef4444; }
.summary-card.expense strong.over-budget { color: #dc2626; font-size: 13px; }
.summary-card.balance.positive strong { color: #3b82f6; }
.summary-card.balance.negative strong { color: #ef4444; }

/* 预算 */
.budget-section {
  background: white;
  border-radius: 16px;
  padding: 14px 16px;
  margin: 0 16px 16px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.06);
}
.budget-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
  font-size: 13px;
  font-weight: 600;
  color: #374151;
}
.budget-input-wrap {
  display: flex;
  align-items: center;
  gap: 4px;
  background: #f3f4f6;
  padding: 4px 10px;
  border-radius: 8px;
  font-size: 13px;
}
.budget-input {
  width: 70px;
  border: none;
  background: transparent;
  outline: none;
  font-size: 13px;
  font-weight: 600;
  color: #667eea;
  text-align: right;
}
.budget-bar {
  height: 10px;
  background: #f3f4f6;
  border-radius: 10px;
  overflow: hidden;
  margin-bottom: 8px;
}
.budget-fill {
  height: 100%;
  background: linear-gradient(90deg, #667eea, #764ba2);
  border-radius: 10px;
  transition: width 0.5s ease;
  min-width: 4px;
}
.budget-fill.over { background: linear-gradient(90deg, #ef4444, #dc2626); }
.budget-info {
  font-size: 12px;
  color: #9ca3af;
  text-align: center;
}
.over-text { color: #ef4444; font-weight: 600; }

/* 平均支出 */
.avg-section {
  display: flex;
  gap: 10px;
  margin: 0 16px 16px;
}
.avg-item {
  flex: 1;
  background: white;
  border-radius: 16px;
  padding: 14px;
  text-align: center;
  box-shadow: 0 2px 8px rgba(0,0,0,0.06);
  transition: transform 0.2s, box-shadow 0.2s;
}
.avg-item:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 16px rgba(0,0,0,0.1);
}
.avg-item .label { font-size: 11px; color: #9ca3af; display: block; margin-bottom: 4px; font-weight: 500; text-transform: uppercase; letter-spacing: 0.5px; }
.avg-item .value { font-size: 20px; font-weight: 700; color: #1a1a2e; }

/* 表单 */
.form-section {
  background: white;
  border-radius: 20px;
  padding: 20px;
  margin: 0 16px 16px;
  box-shadow: 0 2px 12px rgba(0,0,0,0.06);
}
.form-section h2 {
  font-size: 15px;
  font-weight: 600;
  margin-bottom: 14px;
  color: #374151;
  display: flex;
  align-items: center;
  gap: 6px;
}

.type-toggle {
  display: flex;
  gap: 8px;
  margin-bottom: 14px;
  background: #f3f4f6;
  padding: 4px;
  border-radius: 12px;
}
.type-toggle button {
  flex: 1;
  padding: 9px;
  border: none;
  border-radius: 9px;
  cursor: pointer;
  background: transparent;
  color: #9ca3af;
  font-size: 14px;
  font-weight: 600;
  transition: all 0.25s;
}
.type-toggle button.active {
  color: white;
  box-shadow: 0 2px 8px rgba(0,0,0,0.15);
}
.type-toggle button:first-child.active { background: #ef4444; }
.type-toggle button:last-child.active { background: #22c55e; }

.record-form { display: flex; flex-direction: column; gap: 10px; }
.form-row { display: flex; gap: 8px; }
.form-row input, .form-row select {
  flex: 1;
  padding: 12px 14px;
  border: 2px solid #f3f4f6;
  border-radius: 12px;
  font-size: 14px;
  transition: border-color 0.2s;
  background: #fafafa;
  outline: none;
  color: #1a1a2e;
  font-family: inherit;
}
.form-row input:focus, .form-row select:focus {
  border-color: #667eea;
  background: white;
}
.amount-input { flex: 1 !important; }

.submit-btn {
  padding: 14px;
  border: none;
  border-radius: 12px;
  font-size: 16px;
  font-weight: 700;
  cursor: pointer;
  color: white;
  letter-spacing: 0.3px;
  transition: transform 0.15s, box-shadow 0.15s, opacity 0.15s;
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
}
.submit-btn.expense { background: linear-gradient(135deg, #ef4444, #dc2626); }
.submit-btn.income { background: linear-gradient(135deg, #22c55e, #16a34a); }
.submit-btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 6px 16px rgba(0,0,0,0.2);
}
.submit-btn:active { transform: translateY(0); opacity: 0.9; }

.form-actions {
  display: flex;
  gap: 8px;
}
.cancel-btn {
  padding: 14px;
  border: 2px solid #e5e7eb;
  border-radius: 12px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  color: #9ca3af;
  background: transparent;
  transition: all 0.15s;
  flex-shrink: 0;
}
.cancel-btn:hover { border-color: #d1d5db; color: #6b7280; }

.edit-btn {
  background: none;
  border: none;
  color: #d1d5db;
  font-size: 16px;
  cursor: pointer;
  padding: 0 2px;
  transition: color 0.15s;
  line-height: 1;
}
.edit-btn:hover { color: #667eea; }

/* 分类统计 */
.category-section {
  background: white;
  border-radius: 20px;
  padding: 20px;
  margin: 0 16px 16px;
  box-shadow: 0 2px 12px rgba(0,0,0,0.06);
}
.category-section h2 {
  font-size: 15px;
  font-weight: 600;
  margin-bottom: 14px;
  color: #374151;
}
.chart-wrapper {
  max-width: 200px;
  margin: 0 auto 16px;
}
.category-list { display: flex; flex-direction: column; gap: 4px; }
.category-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 12px;
  border-radius: 10px;
  transition: background 0.15s;
}
.category-item:hover { background: #f9fafb; }
.cat-left { display: flex; align-items: center; gap: 8px; }
.cat-icon { font-size: 18px; }
.cat-amount { font-weight: 600; font-size: 14px; color: #4b5563; }

/* 记录列表 */
.records-section {
  background: white;
  border-radius: 20px;
  padding: 20px;
  margin: 0 16px;
  box-shadow: 0 2px 12px rgba(0,0,0,0.06);
}
.records-section h2 {
  font-size: 15px;
  font-weight: 600;
  margin-bottom: 14px;
  color: #374151;
}
.empty {
  text-align: center;
  color: #d1d5db;
  padding: 28px;
  font-size: 14px;
}

.record-list { display: flex; flex-direction: column; gap: 8px; }
.record-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 14px;
  border-radius: 14px;
  background: #f9fafb;
  transition: background 0.15s, transform 0.15s;
}
.record-item:hover { background: #f3f4f6; transform: translateX(2px); }
.record-left { display: flex; align-items: center; gap: 12px; }
.record-icon { font-size: 22px; }
.record-left-text { display: flex; flex-direction: column; gap: 2px; }
.record-cat { font-weight: 600; font-size: 14px; color: #1a1a2e; }
.record-note { font-size: 12px; color: #9ca3af; }
.record-right { display: flex; align-items: center; gap: 10px; }
.record-amount { font-weight: 700; font-size: 16px; }
.record-amount.income { color: #22c55e; }
.record-amount.expense { color: #ef4444; }
.delete-btn {
  background: none;
  border: none;
  color: #d1d5db;
  font-size: 20px;
  cursor: pointer;
  padding: 0 4px;
  transition: color 0.15s;
  line-height: 1;
}
.delete-btn:hover { color: #ef4444; }

/* section 标题通用样式 */
h2::before {
  content: '●';
  font-size: 10px;
  color: #667eea;
}
</style>
