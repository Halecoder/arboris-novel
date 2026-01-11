<template>
  <div class="emotion-curve-section">
    <!-- Header -->
    <div class="flex items-center justify-between mb-6">
      <div class="flex items-center gap-3">
        <div class="w-10 h-10 rounded-full flex items-center justify-center" style="background-color: var(--md-primary-container);">
          <svg class="w-5 h-5" style="color: var(--md-on-primary-container);" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path stroke-linecap="round" stroke-linejoin="round" d="M7 12l3-3 3 3 4-4M8 21l4-4 4 4M3 4h18M4 4h16v12a1 1 0 01-1 1H5a1 1 0 01-1-1V4z" />
          </svg>
        </div>
        <div>
          <h3 class="md-title-medium" style="color: var(--md-on-surface);">情感曲线</h3>
          <p class="md-body-small" style="color: var(--md-on-surface-variant);">追踪章节情感变化</p>
        </div>
      </div>
      <button 
        @click="refreshData" 
        class="md-icon-btn md-ripple"
        :disabled="isLoading"
      >
        <svg 
          class="w-5 h-5 transition-transform" 
          :class="{ 'animate-spin': isLoading }"
          viewBox="0 0 24 24" 
          fill="none" 
          stroke="currentColor" 
          stroke-width="2"
        >
          <path stroke-linecap="round" stroke-linejoin="round" d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15" />
        </svg>
      </button>
    </div>

    <!-- Loading State -->
    <div v-if="isLoading" class="flex flex-col items-center justify-center py-12">
      <div class="md-spinner"></div>
      <p class="mt-4 md-body-medium" style="color: var(--md-on-surface-variant);">分析情感数据中...</p>
    </div>

    <!-- Error State -->
    <div v-else-if="error" class="flex flex-col items-center justify-center py-12">
      <div class="w-12 h-12 rounded-full flex items-center justify-center mb-4" style="background-color: var(--md-error-container);">
        <svg class="w-6 h-6" style="color: var(--md-error);" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path stroke-linecap="round" stroke-linejoin="round" d="M12 8v4m0 4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
        </svg>
      </div>
      <p class="md-body-medium" style="color: var(--md-error);">{{ error }}</p>
      <button @click="refreshData" class="md-btn md-btn-text md-ripple mt-4">重试</button>
    </div>

    <!-- Empty State -->
    <div v-else-if="!emotionData || emotionData.length === 0" class="flex flex-col items-center justify-center py-12">
      <div class="w-16 h-16 rounded-full flex items-center justify-center mb-4" style="background-color: var(--md-surface-container);">
        <svg class="w-8 h-8" style="color: var(--md-on-surface-variant);" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path stroke-linecap="round" stroke-linejoin="round" d="M9 19v-6a2 2 0 00-2-2H5a2 2 0 00-2 2v6a2 2 0 002 2h2a2 2 0 002-2zm0 0V9a2 2 0 012-2h2a2 2 0 012 2v10m-6 0a2 2 0 002 2h2a2 2 0 002-2m0 0V5a2 2 0 012-2h2a2 2 0 012 2v14a2 2 0 01-2 2h-2a2 2 0 01-2-2z" />
        </svg>
      </div>
      <p class="md-body-large" style="color: var(--md-on-surface);">暂无情感数据</p>
      <p class="md-body-medium" style="color: var(--md-on-surface-variant);">生成章节内容后将自动分析情感曲线</p>
    </div>

    <!-- Chart Container -->
    <div v-else>
      <!-- Emotion Type Filter Chips -->
      <div class="flex flex-wrap gap-2 mb-6">
        <button
          v-for="emotion in emotionTypes"
          :key="emotion.key"
          @click="toggleEmotion(emotion.key)"
          class="md-chip md-chip-filter md-ripple"
          :class="{ 'selected': selectedEmotions.includes(emotion.key) }"
          :style="selectedEmotions.includes(emotion.key) ? { backgroundColor: emotion.color + '20', color: emotion.color, borderColor: emotion.color } : {}"
        >
          <span class="w-2 h-2 rounded-full" :style="{ backgroundColor: emotion.color }"></span>
          {{ emotion.label }}
        </button>
      </div>

      <!-- Chart -->
      <div class="md-card md-card-outlined p-4" style="border-radius: var(--md-radius-md);">
        <canvas ref="chartCanvas" height="300"></canvas>
      </div>

      <!-- Legend -->
      <div class="mt-4 flex flex-wrap gap-4">
        <div 
          v-for="emotion in emotionTypes.filter(e => selectedEmotions.includes(e.key))" 
          :key="emotion.key"
          class="flex items-center gap-2"
        >
          <span class="w-3 h-3 rounded-full" :style="{ backgroundColor: emotion.color }"></span>
          <span class="md-label-medium" style="color: var(--md-on-surface-variant);">{{ emotion.label }}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, watch, nextTick } from 'vue'
import { useRoute } from 'vue-router'

interface EmotionDataPoint {
  chapter: number
  joy: number
  sadness: number
  anger: number
  fear: number
  surprise: number
  tension: number
}

const route = useRoute()
const projectId = route.params.id as string

const chartCanvas = ref<HTMLCanvasElement | null>(null)
const isLoading = ref(false)
const error = ref<string | null>(null)
const emotionData = ref<EmotionDataPoint[]>([])
let chartInstance: any = null

const emotionTypes = [
  { key: 'joy', label: '喜悦', color: '#34A853' },
  { key: 'sadness', label: '悲伤', color: '#4285F4' },
  { key: 'anger', label: '愤怒', color: '#EA4335' },
  { key: 'fear', label: '恐惧', color: '#9334E6' },
  { key: 'surprise', label: '惊讶', color: '#FBBC04' },
  { key: 'tension', label: '紧张', color: '#FF6D01' }
]

const selectedEmotions = ref(['joy', 'sadness', 'tension'])

const toggleEmotion = (key: string) => {
  const index = selectedEmotions.value.indexOf(key)
  if (index > -1) {
    if (selectedEmotions.value.length > 1) {
      selectedEmotions.value.splice(index, 1)
    }
  } else {
    selectedEmotions.value.push(key)
  }
  updateChart()
}

const fetchEmotionData = async () => {
  isLoading.value = true
  error.value = null
  
  try {
    const response = await fetch(`/api/analytics/emotion/${projectId}`)
    if (!response.ok) {
      throw new Error('获取情感数据失败')
    }
    const data = await response.json()
    emotionData.value = data.emotions || []
    await nextTick()
    initChart()
  } catch (e: any) {
    error.value = e.message || '加载失败'
  } finally {
    isLoading.value = false
  }
}

const refreshData = () => {
  fetchEmotionData()
}

const initChart = async () => {
  if (!chartCanvas.value || emotionData.value.length === 0) return
  
  // Dynamically import Chart.js
  const { Chart, registerables } = await import('chart.js')
  Chart.register(...registerables)
  
  if (chartInstance) {
    chartInstance.destroy()
  }
  
  const ctx = chartCanvas.value.getContext('2d')
  if (!ctx) return
  
  const labels = emotionData.value.map(d => `第${d.chapter}章`)
  const datasets = emotionTypes
    .filter(e => selectedEmotions.value.includes(e.key))
    .map(emotion => ({
      label: emotion.label,
      data: emotionData.value.map(d => d[emotion.key as keyof EmotionDataPoint] as number),
      borderColor: emotion.color,
      backgroundColor: emotion.color + '20',
      tension: 0.4,
      fill: false,
      pointRadius: 4,
      pointHoverRadius: 6
    }))
  
  chartInstance = new Chart(ctx, {
    type: 'line',
    data: { labels, datasets },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      interaction: {
        intersect: false,
        mode: 'index'
      },
      plugins: {
        legend: {
          display: false
        },
        tooltip: {
          backgroundColor: 'rgba(32, 33, 36, 0.9)',
          titleFont: { family: 'Roboto', size: 14 },
          bodyFont: { family: 'Roboto', size: 12 },
          padding: 12,
          cornerRadius: 8
        }
      },
      scales: {
        x: {
          grid: {
            color: 'rgba(218, 220, 224, 0.5)'
          },
          ticks: {
            font: { family: 'Roboto', size: 12 },
            color: '#5F6368'
          }
        },
        y: {
          min: 0,
          max: 10,
          grid: {
            color: 'rgba(218, 220, 224, 0.5)'
          },
          ticks: {
            font: { family: 'Roboto', size: 12 },
            color: '#5F6368',
            stepSize: 2
          }
        }
      }
    }
  })
}

const updateChart = () => {
  if (chartInstance && emotionData.value.length > 0) {
    const datasets = emotionTypes
      .filter(e => selectedEmotions.value.includes(e.key))
      .map(emotion => ({
        label: emotion.label,
        data: emotionData.value.map(d => d[emotion.key as keyof EmotionDataPoint] as number),
        borderColor: emotion.color,
        backgroundColor: emotion.color + '20',
        tension: 0.4,
        fill: false,
        pointRadius: 4,
        pointHoverRadius: 6
      }))
    
    chartInstance.data.datasets = datasets
    chartInstance.update()
  }
}

onMounted(() => {
  fetchEmotionData()
})

watch(selectedEmotions, () => {
  updateChart()
}, { deep: true })
</script>
