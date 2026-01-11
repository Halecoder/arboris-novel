<template>
  <div class="foreshadowing-section">
    <!-- Header -->
    <div class="flex items-center justify-between mb-6">
      <div class="flex items-center gap-3">
        <div class="w-10 h-10 rounded-full flex items-center justify-center" style="background-color: var(--md-warning-container);">
          <svg class="w-5 h-5" style="color: var(--md-on-warning-container);" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path stroke-linecap="round" stroke-linejoin="round" d="M13 10V3L4 14h7v7l9-11h-7z" />
          </svg>
        </div>
        <div>
          <h3 class="md-title-medium" style="color: var(--md-on-surface);">伏笔管理</h3>
          <p class="md-body-small" style="color: var(--md-on-surface-variant);">追踪故事线索与回收</p>
        </div>
      </div>
      <div class="flex items-center gap-2">
        <button 
          @click="showAddModal = true" 
          class="md-btn md-btn-tonal md-ripple"
        >
          <svg class="w-5 h-5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path stroke-linecap="round" stroke-linejoin="round" d="M12 4v16m8-8H4" />
          </svg>
          添加伏笔
        </button>
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
    </div>

    <!-- Status Filter Tabs -->
    <div class="md-tabs mb-6">
      <button 
        v-for="tab in statusTabs" 
        :key="tab.key"
        @click="activeTab = tab.key"
        class="md-tab"
        :class="{ 'active': activeTab === tab.key }"
      >
        {{ tab.label }}
        <span 
          v-if="getCountByStatus(tab.key) > 0"
          class="ml-2 px-2 py-0.5 rounded-full md-label-small"
          :style="{ backgroundColor: tab.color + '20', color: tab.color }"
        >
          {{ getCountByStatus(tab.key) }}
        </span>
      </button>
    </div>

    <!-- Loading State -->
    <div v-if="isLoading" class="flex flex-col items-center justify-center py-12">
      <div class="md-spinner"></div>
      <p class="mt-4 md-body-medium" style="color: var(--md-on-surface-variant);">加载伏笔数据中...</p>
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
    <div v-else-if="filteredForeshadowing.length === 0" class="flex flex-col items-center justify-center py-12">
      <div class="w-16 h-16 rounded-full flex items-center justify-center mb-4" style="background-color: var(--md-surface-container);">
        <svg class="w-8 h-8" style="color: var(--md-on-surface-variant);" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path stroke-linecap="round" stroke-linejoin="round" d="M13 10V3L4 14h7v7l9-11h-7z" />
        </svg>
      </div>
      <p class="md-body-large" style="color: var(--md-on-surface);">
        {{ activeTab === 'all' ? '暂无伏笔记录' : `暂无${statusTabs.find(t => t.key === activeTab)?.label}的伏笔` }}
      </p>
      <p class="md-body-medium" style="color: var(--md-on-surface-variant);">点击"添加伏笔"开始记录故事线索</p>
    </div>

    <!-- Foreshadowing List -->
    <div v-else class="space-y-4">
      <div 
        v-for="item in filteredForeshadowing" 
        :key="item.id"
        class="md-card md-card-outlined p-4 transition-all duration-200 hover:shadow-md"
        style="border-radius: var(--md-radius-md);"
      >
        <div class="flex items-start justify-between">
          <div class="flex-1">
            <!-- Title & Status -->
            <div class="flex items-center gap-3 mb-2">
              <h4 class="md-title-small" style="color: var(--md-on-surface);">{{ item.title }}</h4>
              <span 
                class="md-chip md-chip-filter selected px-2 py-1"
                :style="{ backgroundColor: getStatusColor(item.status) + '20', color: getStatusColor(item.status) }"
              >
                {{ getStatusLabel(item.status) }}
              </span>
            </div>
            
            <!-- Description -->
            <p class="md-body-medium mb-3" style="color: var(--md-on-surface-variant);">{{ item.description }}</p>
            
            <!-- Metadata -->
            <div class="flex flex-wrap gap-4">
              <div class="flex items-center gap-1">
                <svg class="w-4 h-4" style="color: var(--md-on-surface-variant);" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                  <path stroke-linecap="round" stroke-linejoin="round" d="M12 6.253v13m0-13C10.832 5.477 9.246 5 7.5 5S4.168 5.477 3 6.253v13C4.168 18.477 5.754 18 7.5 18s3.332.477 4.5 1.253m0-13C13.168 5.477 14.754 5 16.5 5c1.747 0 3.332.477 4.5 1.253v13C19.832 18.477 18.247 18 16.5 18c-1.746 0-3.332.477-4.5 1.253" />
                </svg>
                <span class="md-body-small" style="color: var(--md-on-surface-variant);">
                  埋设于第{{ item.planted_chapter }}章
                </span>
              </div>
              <div v-if="item.resolved_chapter" class="flex items-center gap-1">
                <svg class="w-4 h-4" style="color: var(--md-success);" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                  <path stroke-linecap="round" stroke-linejoin="round" d="M5 13l4 4L19 7" />
                </svg>
                <span class="md-body-small" style="color: var(--md-success);">
                  回收于第{{ item.resolved_chapter }}章
                </span>
              </div>
            </div>
          </div>
          
          <!-- Actions -->
          <div class="flex items-center gap-1 ml-4">
            <button 
              v-if="item.status !== 'resolved'"
              @click="markAsResolved(item)"
              class="md-icon-btn md-ripple"
              title="标记为已回收"
            >
              <svg class="w-5 h-5" style="color: var(--md-success);" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path stroke-linecap="round" stroke-linejoin="round" d="M5 13l4 4L19 7" />
              </svg>
            </button>
            <button 
              @click="editItem(item)"
              class="md-icon-btn md-ripple"
              title="编辑"
            >
              <svg class="w-5 h-5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path stroke-linecap="round" stroke-linejoin="round" d="M15.232 5.232l3.536 3.536m-2.036-5.036a2.5 2.5 0 113.536 3.536L6.5 21.036H3v-3.572L16.732 3.732z" />
              </svg>
            </button>
            <button 
              @click="deleteItem(item)"
              class="md-icon-btn md-ripple"
              title="删除"
            >
              <svg class="w-5 h-5" style="color: var(--md-error);" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path stroke-linecap="round" stroke-linejoin="round" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
              </svg>
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- Add/Edit Modal -->
    <transition
      enter-active-class="transition-opacity duration-200"
      leave-active-class="transition-opacity duration-200"
      enter-from-class="opacity-0"
      leave-to-class="opacity-0"
    >
      <div v-if="showAddModal" class="md-dialog-overlay" @click.self="closeModal">
        <div class="md-dialog w-full max-w-lg mx-4">
          <div class="md-dialog-header">
            <h3 class="md-dialog-title">{{ editingItem ? '编辑伏笔' : '添加伏笔' }}</h3>
          </div>
          <div class="md-dialog-content space-y-4">
            <div class="md-text-field">
              <label class="md-text-field-label">伏笔标题</label>
              <input 
                v-model="formData.title" 
                type="text" 
                class="md-text-field-input"
                placeholder="例如：神秘信件"
              />
            </div>
            <div class="md-text-field">
              <label class="md-text-field-label">描述</label>
              <textarea 
                v-model="formData.description" 
                class="md-textarea w-full"
                rows="3"
                placeholder="描述这个伏笔的内容和意义..."
              ></textarea>
            </div>
            <div class="grid grid-cols-2 gap-4">
              <div class="md-text-field">
                <label class="md-text-field-label">埋设章节</label>
                <input 
                  v-model.number="formData.planted_chapter" 
                  type="number" 
                  min="1"
                  class="md-text-field-input"
                  placeholder="1"
                />
              </div>
              <div class="md-text-field">
                <label class="md-text-field-label">回收章节（可选）</label>
                <input 
                  v-model.number="formData.resolved_chapter" 
                  type="number" 
                  min="1"
                  class="md-text-field-input"
                  placeholder="留空表示未回收"
                />
              </div>
            </div>
          </div>
          <div class="md-dialog-actions">
            <button @click="closeModal" class="md-btn md-btn-text md-ripple">取消</button>
            <button @click="saveItem" class="md-btn md-btn-filled md-ripple">保存</button>
          </div>
        </div>
      </div>
    </transition>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { useRoute } from 'vue-router'

interface Foreshadowing {
  id: string
  title: string
  description: string
  planted_chapter: number
  resolved_chapter?: number
  status: 'planted' | 'hinted' | 'resolved'
}

const route = useRoute()
const projectId = route.params.id as string

const isLoading = ref(false)
const error = ref<string | null>(null)
const foreshadowingList = ref<Foreshadowing[]>([])
const activeTab = ref('all')
const showAddModal = ref(false)
const editingItem = ref<Foreshadowing | null>(null)

const formData = ref({
  title: '',
  description: '',
  planted_chapter: 1,
  resolved_chapter: undefined as number | undefined
})

const statusTabs = [
  { key: 'all', label: '全部', color: '#5F6368' },
  { key: 'planted', label: '已埋设', color: '#FBBC04' },
  { key: 'hinted', label: '已暗示', color: '#4285F4' },
  { key: 'resolved', label: '已回收', color: '#34A853' }
]

const filteredForeshadowing = computed(() => {
  if (activeTab.value === 'all') {
    return foreshadowingList.value
  }
  return foreshadowingList.value.filter(item => item.status === activeTab.value)
})

const getCountByStatus = (status: string) => {
  if (status === 'all') return foreshadowingList.value.length
  return foreshadowingList.value.filter(item => item.status === status).length
}

const getStatusColor = (status: string) => {
  const tab = statusTabs.find(t => t.key === status)
  return tab?.color || '#5F6368'
}

const getStatusLabel = (status: string) => {
  const tab = statusTabs.find(t => t.key === status)
  return tab?.label || status
}

const fetchData = async () => {
  isLoading.value = true
  error.value = null
  
  try {
    const response = await fetch(`/api/analytics/foreshadowing/${projectId}`)
    if (!response.ok) {
      throw new Error('获取伏笔数据失败')
    }
    const data = await response.json()
    foreshadowingList.value = data.foreshadowing || []
  } catch (e: any) {
    error.value = e.message || '加载失败'
  } finally {
    isLoading.value = false
  }
}

const refreshData = () => {
  fetchData()
}

const closeModal = () => {
  showAddModal.value = false
  editingItem.value = null
  formData.value = {
    title: '',
    description: '',
    planted_chapter: 1,
    resolved_chapter: undefined
  }
}

const editItem = (item: Foreshadowing) => {
  editingItem.value = item
  formData.value = {
    title: item.title,
    description: item.description,
    planted_chapter: item.planted_chapter,
    resolved_chapter: item.resolved_chapter
  }
  showAddModal.value = true
}

const saveItem = async () => {
  if (!formData.value.title.trim()) {
    return
  }
  
  try {
    const method = editingItem.value ? 'PUT' : 'POST'
    const url = editingItem.value 
      ? `/api/analytics/foreshadowing/${projectId}/${editingItem.value.id}`
      : `/api/analytics/foreshadowing/${projectId}`
    
    const status = formData.value.resolved_chapter ? 'resolved' : 'planted'
    
    const response = await fetch(url, {
      method,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        ...formData.value,
        status
      })
    })
    
    if (!response.ok) {
      throw new Error('保存失败')
    }
    
    await fetchData()
    closeModal()
  } catch (e: any) {
    console.error('Save error:', e)
  }
}

const deleteItem = async (item: Foreshadowing) => {
  if (!confirm(`确定要删除伏笔"${item.title}"吗？`)) {
    return
  }
  
  try {
    const response = await fetch(`/api/analytics/foreshadowing/${projectId}/${item.id}`, {
      method: 'DELETE'
    })
    
    if (!response.ok) {
      throw new Error('删除失败')
    }
    
    await fetchData()
  } catch (e: any) {
    console.error('Delete error:', e)
  }
}

const markAsResolved = async (item: Foreshadowing) => {
  const chapter = prompt('请输入回收章节号：')
  if (!chapter) return
  
  const chapterNum = parseInt(chapter)
  if (isNaN(chapterNum) || chapterNum < 1) {
    alert('请输入有效的章节号')
    return
  }
  
  try {
    const response = await fetch(`/api/analytics/foreshadowing/${projectId}/${item.id}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        ...item,
        resolved_chapter: chapterNum,
        status: 'resolved'
      })
    })
    
    if (!response.ok) {
      throw new Error('更新失败')
    }
    
    await fetchData()
  } catch (e: any) {
    console.error('Update error:', e)
  }
}

onMounted(() => {
  fetchData()
})
</script>
