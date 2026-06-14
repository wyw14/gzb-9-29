<template>
  <div>
    <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:24px;">
      <div>
        <h1 style="color:white;margin-bottom:8px;">新品日历</h1>
        <p style="color:rgba(255,255,255,0.8);">按日期浏览新发布的盲盒</p>
      </div>
    </div>

    <div class="card" style="margin-bottom:20px;">
      <div style="display:flex;justify-content:space-between;align-items:center;flex-wrap:wrap;gap:16px;">
        <div style="display:flex;align-items:center;gap:12px;">
          <button class="btn btn-secondary" @click="prevMonth">
            ◀ 上月
          </button>
          <span style="font-size:20px;font-weight:bold;color:#333;">
            {{ currentYear }}年{{ currentMonth + 1 }}月
          </span>
          <button class="btn btn-secondary" @click="nextMonth">
            下月 ▶
          </button>
          <button class="btn btn-primary" @click="goToToday" style="margin-left:8px;">
            今天
          </button>
        </div>
        <div style="display:flex;align-items:center;gap:8px;">
          <span style="color:#666;font-size:14px;">类别筛选：</span>
          <select v-model="selectedCategory" @change="filterItems" style="width:150px;">
            <option value="all">全部类别</option>
            <option v-for="(name, key) in categories" :key="key" :value="key">
              {{ name }}
            </option>
          </select>
        </div>
      </div>
    </div>

    <div class="card" style="margin-bottom:20px;">
      <div class="calendar-header">
        <div v-for="day in weekDays" :key="day" class="calendar-weekday">
          {{ day }}
        </div>
      </div>
      <div class="calendar-grid">
        <div
          v-for="(day, index) in calendarDays"
          :key="index"
          :class="[
            'calendar-day',
            day.isCurrentMonth ? '' : 'other-month',
            day.isToday ? 'today' : '',
            day.items.length > 0 ? 'has-items' : '',
            selectedDate && day.dateStr === selectedDate ? 'selected' : ''
          ]"
          @click="selectDate(day)"
        >
          <div class="day-number">{{ day.day }}</div>
          <div v-if="day.items.length > 0" class="day-items">
            <div
              v-for="item in day.items.slice(0, 3)"
              :key="item.id"
              class="day-item-dot"
              :style="{ background: getCategoryColor(item.category) }"
            ></div>
            <div v-if="day.items.length > 3" class="more-items">
              +{{ day.items.length - 3 }}
            </div>
          </div>
        </div>
      </div>
    </div>

    <div v-if="selectedDate && selectedDayItems.length > 0" class="card">
      <h2 style="margin-bottom:16px;color:#333;">
        {{ selectedDate }} 新增盲盒 ({{ selectedDayItems.length }})
      </h2>
      <div class="grid grid-3">
        <router-link
          v-for="item in selectedDayItems"
          :key="item.id"
          :to="'/item/' + item.id"
          style="text-decoration:none;color:inherit;"
        >
          <div class="card" style="padding:0;overflow:hidden;cursor:pointer;">
            <div style="width:100%;height:160px;overflow:hidden;background:#f0f0f0;position:relative;">
              <img :src="appendAuth(item.image)" alt="盲盒图片"
                   style="width:100%;height:100%;object-fit:cover;"/>
              <div style="position:absolute;top:12px;right:12px;">
                <span :class="item.status === 'available' ? 'badge badge-available' : 'badge badge-exchanged'">
                  {{ item.status === 'available' ? '可交换' : '已交换' }}
                </span>
              </div>
            </div>
            <div style="padding:12px;">
              <h3 style="margin-bottom:6px;font-size:14px;">{{ getCategoryName(item.category) }}</h3>
              <div style="margin-bottom:6px;">
                <span v-for="tag in item.mysteryTags" :key="tag" class="tag" style="font-size:11px;padding:2px 8px;">
                  {{ tag }}
                </span>
              </div>
            </div>
          </div>
        </router-link>
      </div>
    </div>

    <div v-else-if="selectedDate" class="card empty-state">
      <h2>{{ selectedDate }} 暂无新品</h2>
      <p>这一天没有新发布的盲盒</p>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { getItems, appendAuth } from '../api/index.js'
import { userStore } from '../store/user.js'

const items = ref([])
const filteredItems = ref([])
const loading = ref(true)
const currentYear = ref(new Date().getFullYear())
const currentMonth = ref(new Date().getMonth())
const selectedCategory = ref('all')
const selectedDate = ref(null)

const categories = {
  book: '书籍类',
  figure: '手办类',
  toy: '玩具类',
  game: '游戏类',
  digital: '数码类',
  other: '其他'
}

const categoryColors = {
  book: '#f093fb',
  figure: '#667eea',
  toy: '#f5576c',
  game: '#4facfe',
  digital: '#43e97b',
  other: '#fa709a'
}

const weekDays = ['日', '一', '二', '三', '四', '五', '六']

function getCategoryName(key) {
  return categories[key] || key
}

function getCategoryColor(category) {
  return categoryColors[category] || '#999'
}

const itemsByDateMap = computed(() => {
  const map = {}
  filteredItems.value.forEach(item => {
    const dateStr = formatDateKey(new Date(item.createdAt))
    if (!map[dateStr]) {
      map[dateStr] = []
    }
    map[dateStr].push(item)
  })
  return map
})

const calendarDays = computed(() => {
  const year = currentYear.value
  const month = currentMonth.value
  
  const firstDay = new Date(year, month, 1)
  const lastDay = new Date(year, month + 1, 0)
  const daysInMonth = lastDay.getDate()
  const startDayOfWeek = firstDay.getDay()
  
  const prevMonthLastDay = new Date(year, month, 0).getDate()
  
  const days = []
  const today = new Date()
  const todayStr = formatDateKey(today)
  const map = itemsByDateMap.value
  
  for (let i = startDayOfWeek - 1; i >= 0; i--) {
    const day = prevMonthLastDay - i
    const dateStr = formatDateKey(new Date(year, month - 1, day))
    days.push({
      day,
      dateStr,
      isCurrentMonth: false,
      isToday: false,
      items: map[dateStr] || []
    })
  }
  
  for (let day = 1; day <= daysInMonth; day++) {
    const dateStr = formatDateKey(new Date(year, month, day))
    days.push({
      day,
      dateStr,
      isCurrentMonth: true,
      isToday: dateStr === todayStr,
      items: map[dateStr] || []
    })
  }
  
  const remainingDays = 42 - days.length
  for (let day = 1; day <= remainingDays; day++) {
    const dateStr = formatDateKey(new Date(year, month + 1, day))
    days.push({
      day,
      dateStr,
      isCurrentMonth: false,
      isToday: false,
      items: map[dateStr] || []
    })
  }
  
  return days
})

const selectedDayItems = computed(() => {
  if (!selectedDate.value) return []
  return itemsByDateMap.value[selectedDate.value] || []
})

function formatDateKey(date) {
  const y = date.getFullYear()
  const m = String(date.getMonth() + 1).padStart(2, '0')
  const d = String(date.getDate()).padStart(2, '0')
  return `${y}-${m}-${d}`
}

function isDateInCurrentMonth(dateStr) {
  if (!dateStr) return false
  const [y, m] = dateStr.split('-').map(Number)
  return y === currentYear.value && (m - 1) === currentMonth.value
}

function getDaysInMonth(year, month) {
  return new Date(year, month + 1, 0).getDate()
}

function adjustSelectedDate() {
  if (!selectedDate.value) return
  
  if (!isDateInCurrentMonth(selectedDate.value)) {
    const daysInNewMonth = getDaysInMonth(currentYear.value, currentMonth.value)
    const [, , oldDay] = selectedDate.value.split('-').map(Number)
    const newDay = Math.min(oldDay, daysInNewMonth)
    selectedDate.value = formatDateKey(new Date(currentYear.value, currentMonth.value, newDay))
  }
}

function prevMonth() {
  if (currentMonth.value === 0) {
    currentMonth.value = 11
    currentYear.value--
  } else {
    currentMonth.value--
  }
  adjustSelectedDate()
}

function nextMonth() {
  if (currentMonth.value === 11) {
    currentMonth.value = 0
    currentYear.value++
  } else {
    currentMonth.value++
  }
  adjustSelectedDate()
}

function goToToday() {
  const today = new Date()
  currentYear.value = today.getFullYear()
  currentMonth.value = today.getMonth()
  selectedDate.value = formatDateKey(today)
}

function selectDate(day) {
  if (day.isCurrentMonth) {
    selectedDate.value = day.dateStr
  }
}

function filterItems() {
  if (selectedCategory.value === 'all') {
    filteredItems.value = items.value
  } else {
    filteredItems.value = items.value.filter(item => item.category === selectedCategory.value)
  }
}

watch(selectedCategory, () => {
  filterItems()
})

async function loadItems() {
  loading.value = true
  try {
    items.value = await getItems(userStore.user.id)
    filterItems()
  } catch (e) {
    console.error(e)
  } finally {
    loading.value = false
  }
}

onMounted(() => {
  loadItems()
  const today = new Date()
  selectedDate.value = formatDateKey(today)
})
</script>

<style scoped>
.calendar-header {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  border-bottom: 2px solid #e0e0e0;
  margin-bottom: 8px;
}

.calendar-weekday {
  padding: 12px;
  text-align: center;
  font-weight: 600;
  color: #666;
  font-size: 14px;
}

.calendar-grid {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 4px;
}

.calendar-day {
  min-height: 100px;
  padding: 8px;
  border: 1px solid #f0f0f0;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s;
  position: relative;
}

.calendar-day:hover {
  background: #f8f9ff;
  border-color: #667eea;
}

.calendar-day.other-month {
  color: #ccc;
  background: #fafafa;
  cursor: default;
}

.calendar-day.other-month:hover {
  background: #fafafa;
  border-color: #f0f0f0;
}

.calendar-day.today {
  background: linear-gradient(135deg, #667eea20 0%, #764ba220 100%);
  border-color: #667eea;
}

.calendar-day.today .day-number {
  color: #667eea;
  font-weight: bold;
}

.calendar-day.selected {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-color: transparent;
}

.calendar-day.selected .day-number {
  color: white;
}

.calendar-day.has-items {
  border-color: #d4edda;
}

.day-number {
  font-size: 16px;
  margin-bottom: 4px;
}

.day-items {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
  align-items: center;
}

.day-item-dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
}

.more-items {
  font-size: 11px;
  color: #999;
}

.calendar-day.selected .more-items {
  color: rgba(255, 255, 255, 0.8);
}
</style>
