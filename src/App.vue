<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'

interface HistoryItem {
  id: string
  name: string
  monthlySalary: number
  workDaysPerMonth: number
  workStartTime: string
  workEndTime: string
  lunchStart: string
  lunchEnd: string
  createdAt: string
}

interface CurrentConfig {
  mode: 'hourly' | 'monthly'
  hourlyWage: number
  monthlySalary: number
  workDaysPerMonth: number
  workStartTime: string
  workEndTime: string
  lunchStart: string
  lunchEnd: string
  isDarkMode: boolean
}

const mode = ref<'hourly' | 'monthly'>('monthly')
const hourlyWage = ref(50)
const monthlySalary = ref(10000)
const workDaysPerMonth = ref(22)

const workStartTime = ref('09:00')
const workEndTime = ref('18:00')
const lunchStart = ref('12:00')
const lunchEnd = ref('13:00')

const currentTime = ref(new Date())
const isDarkMode = ref(false)
const showHistory = ref(false)
const historyName = ref('')
const historyItems = ref<HistoryItem[]>([])

let timer: ReturnType<typeof setInterval> | null = null

onMounted(() => {
  timer = setInterval(() => {
    currentTime.value = new Date()
  }, 1000)
  
  loadSettings()
  loadHistory()
  calculateHourlyFromMonthly()
})

onUnmounted(() => {
  if (timer) {
    clearInterval(timer)
  }
})

watch([monthlySalary, workDaysPerMonth], () => {
  if (mode.value === 'monthly') {
    calculateHourlyFromMonthly()
  }
})

watch(isDarkMode, (newVal) => {
  if (newVal) {
    document.documentElement.classList.add('dark')
  } else {
    document.documentElement.classList.remove('dark')
  }
  autoSaveConfig()
})

watch([mode, hourlyWage, monthlySalary, workDaysPerMonth, workStartTime, workEndTime, lunchStart, lunchEnd], () => {
  autoSaveConfig()
})

const loadSettings = () => {
  const savedConfig = localStorage.getItem('salary-calc-config')
  if (savedConfig) {
    try {
      const config: CurrentConfig = JSON.parse(savedConfig)
      mode.value = config.mode || 'monthly'
      hourlyWage.value = config.hourlyWage || 50
      monthlySalary.value = config.monthlySalary || 10000
      workDaysPerMonth.value = config.workDaysPerMonth || 22
      workStartTime.value = config.workStartTime || '09:00'
      workEndTime.value = config.workEndTime || '18:00'
      lunchStart.value = config.lunchStart || '12:00'
      lunchEnd.value = config.lunchEnd || '13:00'
      isDarkMode.value = config.isDarkMode || false
      
      if (isDarkMode.value) {
        document.documentElement.classList.add('dark')
      }
    } catch {
      console.error('Failed to load saved config')
    }
  }
}

const autoSaveConfig = () => {
  const config: CurrentConfig = {
    mode: mode.value,
    hourlyWage: hourlyWage.value,
    monthlySalary: monthlySalary.value,
    workDaysPerMonth: workDaysPerMonth.value,
    workStartTime: workStartTime.value,
    workEndTime: workEndTime.value,
    lunchStart: lunchStart.value,
    lunchEnd: lunchEnd.value,
    isDarkMode: isDarkMode.value
  }
  localStorage.setItem('salary-calc-config', JSON.stringify(config))
}

const loadHistory = () => {
  const saved = localStorage.getItem('salary-calc-history')
  if (saved) {
    try {
      historyItems.value = JSON.parse(saved)
    } catch {
      historyItems.value = []
    }
  }
}

const saveHistory = () => {
  localStorage.setItem('salary-calc-history', JSON.stringify(historyItems.value))
}

const calculateHourlyFromMonthly = () => {
  const monthly = monthlySalary.value || 0
  const days = workDaysPerMonth.value || 22
  const dailyHours = 8
  hourlyWage.value = Math.round((monthly / days / dailyHours) * 100) / 100
}

const parseTimeToMinutes = (timeStr: string): number => {
  const [hours, minutes] = timeStr.split(':').map(Number)
  return (hours || 0) * 60 + (minutes || 0)
}

const getWorkHoursPerDay = () => {
  const start = parseTimeToMinutes(workStartTime.value)
  const end = parseTimeToMinutes(workEndTime.value)
  const lunchStartMin = parseTimeToMinutes(lunchStart.value)
  const lunchEndMin = parseTimeToMinutes(lunchEnd.value)
  
  const totalMinutes = end - start - (lunchEndMin - lunchStartMin)
  return Math.max(0, totalMinutes) / 60
}

const formatTime = (date: Date): string => {
  const hours = date.getHours().toString().padStart(2, '0')
  const minutes = date.getMinutes().toString().padStart(2, '0')
  const seconds = date.getSeconds().toString().padStart(2, '0')
  return `${hours}:${minutes}:${seconds}`
}

const getCurrentWorkMinutes = computed(() => {
  const now = currentTime.value
  const nowMinutes = now.getHours() * 60 + now.getMinutes()
  const nowSeconds = now.getSeconds()
  
  const workStart = parseTimeToMinutes(workStartTime.value)
  const workEnd = parseTimeToMinutes(workEndTime.value)
  const lunchStartMin = parseTimeToMinutes(lunchStart.value)
  const lunchEndMin = parseTimeToMinutes(lunchEnd.value)
  
  if (nowMinutes < workStart || nowMinutes >= workEnd) {
    return 0
  }
  
  let workedMinutes = nowMinutes - workStart
  
  if (nowMinutes >= lunchEndMin) {
    workedMinutes -= (lunchEndMin - lunchStartMin)
  } else if (nowMinutes >= lunchStartMin) {
    workedMinutes -= (nowMinutes - lunchStartMin)
  }
  
  return Math.max(0, workedMinutes) + nowSeconds / 60
})

const workedHours = computed(() => {
  return getCurrentWorkMinutes.value / 60
})

const todayEarnings = computed(() => {
  return workedHours.value * hourlyWage.value
})

const workProgress = computed(() => {
  const totalMinutes = getWorkHoursPerDay() * 60
  if (totalMinutes === 0) return 0
  return Math.min(100, (getCurrentWorkMinutes.value / totalMinutes) * 100)
})

const switchMode = (newMode: 'hourly' | 'monthly') => {
  mode.value = newMode
  if (newMode === 'monthly') {
    calculateHourlyFromMonthly()
  } else {
    const dailyHours = getWorkHoursPerDay()
    monthlySalary.value = Math.round(hourlyWage.value * dailyHours * workDaysPerMonth.value)
  }
}

const formatCurrency = (value: number): string => {
  return value.toFixed(2)
}

const toggleDarkMode = () => {
  isDarkMode.value = !isDarkMode.value
}

const addToHistory = () => {
  const name = historyName.value.trim() || `工资配置 ${historyItems.value.length + 1}`
  const item: HistoryItem = {
    id: Date.now().toString(),
    name,
    monthlySalary: monthlySalary.value,
    workDaysPerMonth: workDaysPerMonth.value,
    workStartTime: workStartTime.value,
    workEndTime: workEndTime.value,
    lunchStart: lunchStart.value,
    lunchEnd: lunchEnd.value,
    createdAt: new Date().toLocaleString('zh-CN')
  }
  historyItems.value.unshift(item)
  if (historyItems.value.length > 10) {
    historyItems.value = historyItems.value.slice(0, 10)
  }
  saveHistory()
  historyName.value = ''
  showHistory.value = false
}

const loadFromHistory = (item: HistoryItem) => {
  monthlySalary.value = item.monthlySalary
  workDaysPerMonth.value = item.workDaysPerMonth
  workStartTime.value = item.workStartTime
  workEndTime.value = item.workEndTime
  lunchStart.value = item.lunchStart
  lunchEnd.value = item.lunchEnd
  mode.value = 'monthly'
  showHistory.value = false
}

const deleteFromHistory = (id: string) => {
  historyItems.value = historyItems.value.filter(item => item.id !== id)
  saveHistory()
}
</script>

<template>
  <div class="calculator" :class="{ 'dark-mode': isDarkMode }">
    <div class="header">
      <div class="header-top">
        <h1>💰 实时工资计算器</h1>
        <div class="header-actions">
          <button @click="toggleDarkMode" class="action-btn" :title="isDarkMode ? '切换到浅色模式' : '切换到深色模式'">
            {{ isDarkMode ? '☀️' : '🌙' }}
          </button>
          <button @click="showHistory = !showHistory" class="action-btn" title="历史记录">
            📋
          </button>
        </div>
      </div>
      <div class="current-time">{{ formatTime(currentTime) }}</div>
    </div>

    <div class="earnings-display">
      <div class="earnings-label">今日已挣</div>
      <div class="earnings-amount">¥{{ formatCurrency(todayEarnings) }}</div>
      <div class="earnings-rate">
        时薪 ¥{{ hourlyWage }} | 
        日薪 ¥{{ (hourlyWage * getWorkHoursPerDay()).toFixed(2) }}
      </div>
    </div>

    <div class="progress-section">
      <div class="progress-info">
        <span>工作进度</span>
        <span>{{ workProgress.toFixed(1) }}%</span>
      </div>
      <div class="progress-bar">
        <div class="progress-fill" :style="{ width: workProgress + '%' }"></div>
      </div>
      <div class="time-info">
        <span>🕐 {{ workStartTime }} - {{ workEndTime }}</span>
        <span>🍱 午休 {{ lunchStart }} - {{ lunchEnd }}</span>
      </div>
    </div>

    <div class="mode-switch">
      <button 
        @click="switchMode('monthly')" 
        class="mode-btn"
        :class="{ active: mode === 'monthly' }"
      >
        月薪模式
      </button>
      <button 
        @click="switchMode('hourly')" 
        class="mode-btn"
        :class="{ active: mode === 'hourly' }"
      >
        时薪模式
      </button>
    </div>

    <div class="controls">
      <template v-if="mode === 'monthly'">
        <div class="input-group">
          <label> 月薪 (元)</label>
          <input 
            v-model.number="monthlySalary" 
            type="number" 
            min="0" 
            step="100"
            class="salary-input"
          />
        </div>
        <div class="input-row">
          <div class="input-group small">
            <label>每月工作天数</label>
            <input 
              v-model.number="workDaysPerMonth" 
              type="number" 
              min="1" 
              max="31"
              class="small-input"
            />
          </div>
          <div class="input-group small">
            <label>日工作时长</label>
            <div class="time-display-static">
              {{ getWorkHoursPerDay().toFixed(1) }} 小时
            </div>
          </div>
        </div>
        <div class="conversion-info">
           时薪 = ¥{{ monthlySalary }} ÷ {{ workDaysPerMonth }}天 ÷ 8小时 = ¥{{ hourlyWage }}/小时
        </div>
      </template>

      <template v-else>
        <div class="input-group">
          <label>💰 时薪 (元)</label>
          <input 
            v-model.number="hourlyWage" 
            type="number" 
            min="0" 
            step="0.5"
            class="hourly-input"
          />
        </div>
        <div class="conversion-info">
          💡 日薪 = ¥{{ hourlyWage }} × {{ getWorkHoursPerDay().toFixed(1) }}小时 = ¥{{ (hourlyWage * getWorkHoursPerDay()).toFixed(2) }}
        </div>
      </template>

      <div class="work-time-settings">
        <h3>⏰ 工作时间设置</h3>
        <div class="input-row">
          <div class="input-group small">
            <label>上班时间</label>
            <input 
              v-model="workStartTime" 
              type="time" 
              class="small-input"
            />
          </div>
          <div class="input-group small">
            <label>下班时间</label>
            <input 
              v-model="workEndTime" 
              type="time" 
              class="small-input"
            />
          </div>
        </div>
        <div class="input-row">
          <div class="input-group small">
            <label>午休开始</label>
            <input 
              v-model="lunchStart" 
              type="time" 
              class="small-input"
            />
          </div>
          <div class="input-group small">
            <label>午休结束</label>
            <input 
              v-model="lunchEnd" 
              type="time" 
              class="small-input"
            />
          </div>
        </div>
      </div>

      <div class="history-section" v-if="showHistory">
        <h3>📋 历史记录</h3>
        
        <div v-if="historyItems.length === 0" class="empty-history">
          暂无历史记录，点击下方按钮保存当前配置
        </div>
        
        <div v-else class="history-list">
          <div 
            v-for="item in historyItems" 
            :key="item.id" 
            class="history-item"
          >
            <div class="history-item-header">
              <span class="history-name">{{ item.name }}</span>
              <button 
                @click="deleteFromHistory(item.id)" 
                class="delete-btn"
                title="删除"
              >
                ✕
              </button>
            </div>
            <div class="history-item-details">
              <span>月薪: ¥{{ item.monthlySalary }}</span>
              <span>工作天数: {{ item.workDaysPerMonth }}天</span>
              <span>工作时间: {{ item.workStartTime }}-{{ item.workEndTime }}</span>
            </div>
            <div class="history-item-footer">
              <span class="history-date">{{ item.createdAt }}</span>
              <button 
                @click="loadFromHistory(item)" 
                class="load-btn"
              >
                加载配置
              </button>
            </div>
          </div>
        </div>

        <div class="save-section">
          <input 
            v-model="historyName" 
            type="text" 
            placeholder="输入配置名称（可选）"
            class="history-name-input"
          />
          <button @click="addToHistory" class="save-btn">
            💾 保存当前配置
          </button>
        </div>
      </div>
    </div>

    <div class="tips">
      💡 提示：所有配置会自动保存，下次打开页面时自动加载
    </div>
  </div>
</template>

<style scoped>
* {
  box-sizing: border-box;
}

.calculator {
  background: white;
  border-radius: 24px;
  padding: 24px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  max-width: 500px;
  margin: 0 auto;
  transition: background-color 0.3s, color 0.3s;
}

@media (max-width: 600px) {
  .calculator {
    padding: 16px;
    border-radius: 0;
    min-height: 100vh;
  }
}

.calculator.dark-mode {
  background: #1a1a2e;
}

.header {
  text-align: center;
  margin-bottom: 20px;
}

.header-top {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.header h1 {
  font-size: 20px;
  color: #333;
  margin: 0;
}

@media (max-width: 600px) {
  .header h1 {
    font-size: 18px;
  }
}

.dark-mode .header h1 {
  color: #fff;
}

.header-actions {
  display: flex;
  gap: 8px;
}

.action-btn {
  width: 40px;
  height: 40px;
  border: none;
  border-radius: 10px;
  background: #f0f0f0;
  font-size: 20px;
  cursor: pointer;
  transition: all 0.3s;
}

@media (max-width: 600px) {
  .action-btn {
    width: 44px;
    height: 44px;
    font-size: 22px;
  }
}

.dark-mode .action-btn {
  background: #2d2d44;
}

.action-btn:hover {
  background: #e0e0e0;
  transform: scale(1.1);
}

.dark-mode .action-btn:hover {
  background: #3d3d54;
}

.current-time {
  font-size: 20px;
  color: #666;
  font-family: 'Courier New', monospace;
}

@media (max-width: 600px) {
  .current-time {
    font-size: 24px;
    font-weight: bold;
  }
}

.dark-mode .current-time {
  color: #aaa;
}

.earnings-display {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-radius: 16px;
  padding: 28px 24px;
  text-align: center;
  margin-bottom: 20px;
}

.earnings-label {
  font-size: 16px;
  color: rgba(255, 255, 255, 0.9);
  margin-bottom: 8px;
}

@media (max-width: 600px) {
  .earnings-label {
    font-size: 18px;
  }
}

.earnings-amount {
  font-size: 48px;
  font-weight: bold;
  color: white;
  margin-bottom: 12px;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}

@media (max-width: 600px) {
  .earnings-amount {
    font-size: 56px;
  }
}

.earnings-rate {
  font-size: 14px;
  color: rgba(255, 255, 255, 0.9);
}

@media (max-width: 600px) {
  .earnings-rate {
    font-size: 16px;
  }
}

.progress-section {
  background: #f8f9fa;
  border-radius: 12px;
  padding: 16px;
  margin-bottom: 20px;
}

.dark-mode .progress-section {
  background: #2d2d44;
}

.progress-info {
  display: flex;
  justify-content: space-between;
  margin-bottom: 12px;
  font-size: 16px;
  color: #666;
  font-weight: 600;
}

@media (max-width: 600px) {
  .progress-info {
    font-size: 18px;
  }
}

.dark-mode .progress-info {
  color: #ccc;
}

.progress-bar {
  background: #e0e0e0;
  border-radius: 10px;
  height: 20px;
  overflow: hidden;
  margin-bottom: 12px;
}

@media (max-width: 600px) {
  .progress-bar {
    height: 24px;
  }
}

.dark-mode .progress-bar {
  background: #3d3d54;
}

.progress-fill {
  background: linear-gradient(90deg, #667eea 0%, #764ba2 100%);
  height: 100%;
  transition: width 0.3s ease;
  border-radius: 10px;
}

.time-info {
  display: flex;
  justify-content: space-between;
  font-size: 14px;
  color: #666;
  flex-wrap: wrap;
  gap: 8px;
}

@media (max-width: 600px) {
  .time-info {
    font-size: 16px;
  }
}

.dark-mode .time-info {
  color: #aaa;
}

.mode-switch {
  display: flex;
  gap: 8px;
  margin-bottom: 16px;
}

.mode-btn {
  flex: 1;
  padding: 12px 16px;
  font-size: 16px;
  border: 2px solid #e0e0e0;
  border-radius: 10px;
  background: white;
  color: #666;
  cursor: pointer;
  transition: all 0.3s;
  font-weight: 600;
}

@media (max-width: 600px) {
  .mode-btn {
    padding: 14px 16px;
    font-size: 17px;
  }
}

.dark-mode .mode-btn {
  background: #2d2d44;
  border-color: #444;
  color: #ccc;
}

.mode-btn:hover {
  border-color: #667eea;
}

.mode-btn.active {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-color: #667eea;
  color: white;
}

.controls {
  margin-bottom: 20px;
}

.input-group {
  margin-bottom: 16px;
}

.input-group label {
  display: block;
  font-size: 16px;
  color: #666;
  margin-bottom: 8px;
  font-weight: 600;
}

@media (max-width: 600px) {
  .input-group label {
    font-size: 17px;
  }
}

.dark-mode .input-group label {
  color: #ccc;
}

.salary-input,
.hourly-input {
  width: 100%;
  padding: 16px;
  font-size: 22px;
  border: 2px solid #e0e0e0;
  border-radius: 12px;
  outline: none;
  transition: border-color 0.3s;
  background: white;
  font-weight: 600;
}

@media (max-width: 600px) {
  .salary-input,
  .hourly-input {
    padding: 18px;
    font-size: 24px;
  }
}

.dark-mode .salary-input,
.dark-mode .hourly-input {
  background: #2d2d44;
  border-color: #444;
  color: #fff;
}

.small-input {
  width: 100%;
  padding: 14px;
  font-size: 18px;
  border: 2px solid #e0e0e0;
  border-radius: 10px;
  outline: none;
  transition: border-color 0.3s;
  background: white;
  font-weight: 600;
}

@media (max-width: 600px) {
  .small-input {
    padding: 16px;
    font-size: 20px;
  }
}

.dark-mode .small-input {
  background: #2d2d44;
  border-color: #444;
  color: #fff;
}

.dark-mode .small-input[type="time"] {
  color-scheme: dark;
}

.salary-input:focus,
.hourly-input:focus,
.small-input:focus {
  border-color: #667eea;
}

.input-row {
  display: flex;
  gap: 12px;
}

.input-group.small {
  flex: 1;
  margin-bottom: 16px;
}

.time-display-static {
  padding: 14px;
  font-size: 18px;
  color: #333;
  font-weight: 600;
  background: #f8f9fa;
  border-radius: 10px;
  text-align: center;
}

@media (max-width: 600px) {
  .time-display-static {
    padding: 16px;
    font-size: 20px;
  }
}

.dark-mode .time-display-static {
  background: #3d3d54;
  color: #fff;
}

.conversion-info {
  background: #e8f4fd;
  border-radius: 10px;
  padding: 14px;
  font-size: 14px;
  color: #31708f;
  margin-bottom: 16px;
  line-height: 1.6;
}

@media (max-width: 600px) {
  .conversion-info {
    font-size: 16px;
    padding: 16px;
  }
}

.dark-mode .conversion-info {
  background: #1e3a5f;
  color: #87ceeb;
}

.work-time-settings {
  background: #f8f9fa;
  border-radius: 12px;
  padding: 16px;
  margin-top: 16px;
}

.dark-mode .work-time-settings {
  background: #2d2d44;
}

.work-time-settings h3 {
  font-size: 16px;
  color: #333;
  margin-bottom: 16px;
}

@media (max-width: 600px) {
  .work-time-settings h3 {
    font-size: 18px;
  }
}

.dark-mode .work-time-settings h3 {
  color: #fff;
}

.history-section {
  background: #f8f9fa;
  border-radius: 12px;
  padding: 16px;
  margin-top: 16px;
}

.dark-mode .history-section {
  background: #2d2d44;
}

.history-section h3 {
  font-size: 16px;
  color: #333;
  margin-bottom: 16px;
}

@media (max-width: 600px) {
  .history-section h3 {
    font-size: 18px;
  }
}

.dark-mode .history-section h3 {
  color: #fff;
}

.empty-history {
  text-align: center;
  color: #999;
  padding: 20px;
  font-size: 14px;
}

@media (max-width: 600px) {
  .empty-history {
    font-size: 16px;
  }
}

.dark-mode .empty-history {
  color: #666;
}

.history-list {
  max-height: 300px;
  overflow-y: auto;
  margin-bottom: 16px;
}

.history-item {
  background: white;
  border-radius: 10px;
  padding: 14px;
  margin-bottom: 10px;
  border: 1px solid #e0e0e0;
}

@media (max-width: 600px) {
  .history-item {
    padding: 16px;
  }
}

.dark-mode .history-item {
  background: #3d3d54;
  border-color: #444;
}

.history-item-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.history-name {
  font-weight: 600;
  color: #333;
  font-size: 15px;
}

@media (max-width: 600px) {
  .history-name {
    font-size: 17px;
  }
}

.dark-mode .history-name {
  color: #fff;
}

.delete-btn {
  width: 28px;
  height: 28px;
  border: none;
  border-radius: 6px;
  background: #ff4444;
  color: white;
  font-size: 14px;
  cursor: pointer;
  transition: background 0.3s;
}

@media (max-width: 600px) {
  .delete-btn {
    width: 32px;
    height: 32px;
    font-size: 16px;
  }
}

.delete-btn:hover {
  background: #cc0000;
}

.history-item-details {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  font-size: 13px;
  color: #666;
  margin-bottom: 10px;
}

@media (max-width: 600px) {
  .history-item-details {
    font-size: 15px;
    gap: 12px;
  }
}

.dark-mode .history-item-details {
  color: #aaa;
}

.history-item-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.history-date {
  font-size: 12px;
  color: #999;
}

@media (max-width: 600px) {
  .history-date {
    font-size: 13px;
  }
}

.dark-mode .history-date {
  color: #666;
}

.load-btn {
  padding: 6px 14px;
  border: none;
  border-radius: 6px;
  background: #667eea;
  color: white;
  font-size: 13px;
  cursor: pointer;
  transition: background 0.3s;
  font-weight: 600;
}

@media (max-width: 600px) {
  .load-btn {
    padding: 8px 16px;
    font-size: 15px;
  }
}

.load-btn:hover {
  background: #5568d3;
}

.save-section {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.history-name-input {
  flex: 1;
  min-width: 150px;
  padding: 12px;
  border: 2px solid #e0e0e0;
  border-radius: 10px;
  font-size: 15px;
  outline: none;
  transition: border-color 0.3s;
}

@media (max-width: 600px) {
  .history-name-input {
    width: 100%;
    padding: 14px;
    font-size: 17px;
  }
}

.dark-mode .history-name-input {
  background: #3d3d54;
  border-color: #444;
  color: #fff;
}

.history-name-input:focus {
  border-color: #667eea;
}

.save-btn {
  padding: 12px 20px;
  border: none;
  border-radius: 10px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  font-size: 14px;
  cursor: pointer;
  transition: transform 0.3s;
  font-weight: 600;
}

@media (max-width: 600px) {
  .save-btn {
    width: 100%;
    padding: 16px;
    font-size: 16px;
  }
}

.save-btn:hover {
  transform: scale(1.02);
}

.tips {
  text-align: center;
  font-size: 13px;
  color: #999;
  line-height: 1.6;
}

@media (max-width: 600px) {
  .tips {
    font-size: 15px;
  }
}

.dark-mode .tips {
  color: #666;
}
</style>