<template>
  <div class="app-shell">
    <div class="ambient" aria-hidden="true">
      <span v-for="particle in particles" :key="particle.id" class="particle" :style="particle.style" />
      <div class="orb orb-violet" />
      <div class="orb orb-amber" />
    </div>

    <header class="navbar">
      <a class="brand" href="#" aria-label="TaskBoard home">
        <span class="brand-mark">
          <svg viewBox="0 0 24 24" aria-hidden="true">
            <path d="M8 5h8M8 10h8M8 15h5" />
            <path d="M6 3h12a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2Z" />
          </svg>
        </span>
        <span>TaskBoard</span>
      </a>

      <div class="nav-meta">
        <span class="live-dot" />
        <span>Workspace online</span>
      </div>

      <button class="primary-button compact" type="button" @click="openCreate">
        <svg viewBox="0 0 24 24" aria-hidden="true"><path d="M12 5v14M5 12h14" /></svg>
        เพิ่มงาน
      </button>
    </header>

    <main>
      <section class="hero-section">
        <div class="hero-copy">
          <span class="eyebrow"><span>●</span> YOUR PRODUCTIVITY OS</span>
          <h1>
            จัดการงานให้ชัด
            <span class="gradient-text">ขยับทุกอย่างให้ไว</span>
          </h1>
          <p>พื้นที่เดียวสำหรับวางแผน ติดตาม และปิดงานของทีม โดยไม่ปล่อยให้งานสำคัญหลุดจากสายตา</p>
          <div class="hero-actions">
            <button class="primary-button" type="button" @click="openCreate">
              สร้างงานใหม่
              <svg viewBox="0 0 24 24" aria-hidden="true"><path d="m9 18 6-6-6-6" /></svg>
            </button>
            <button class="ghost-button" type="button" @click="scrollToBoard">
              ดูบอร์ดงาน
              <span>↓</span>
            </button>
          </div>
        </div>

        <div class="signal-panel">
          <div class="signal-topline">
            <span>LIVE WORKSPACE</span>
            <span>{{ completionRate }}% complete</span>
          </div>
          <div class="big-number">{{ tasks.length.toString().padStart(2, '0') }}</div>
          <p>งานในระบบทั้งหมด</p>
          <div class="progress-track">
            <span :style="{ width: `${completionRate}%` }" />
          </div>
          <div class="signal-stats">
            <div><strong>{{ counts.todo }}</strong><span>รอทำ</span></div>
            <div><strong>{{ counts.inprogress }}</strong><span>กำลังทำ</span></div>
            <div><strong>{{ counts.done }}</strong><span>สำเร็จ</span></div>
          </div>
        </div>
      </section>

      <section ref="boardSection" class="board-section">
        <div class="section-heading">
          <div>
            <span class="section-kicker">WORKFLOW OVERVIEW</span>
            <h2>งานของคุณในมุมมองเดียว</h2>
          </div>
          <span class="task-count">{{ filteredTasks.length }} งานที่แสดง</span>
        </div>

        <div class="toolbar">
          <label class="search-field">
            <svg viewBox="0 0 24 24" aria-hidden="true">
              <circle cx="11" cy="11" r="7" />
              <path d="m20 20-3.6-3.6" />
            </svg>
            <input v-model="searchQuery" type="search" placeholder="ค้นหาชื่องานหรือรายละเอียด..." />
            <button v-if="searchQuery" type="button" aria-label="ล้างคำค้น" @click="searchQuery = ''">×</button>
          </label>

          <div class="filter-group">
            <CustomSelect v-model="filterPriority" :options="priorityFilterOptions" aria-label="กรองตามความสำคัญ" />
            <button
              v-if="searchQuery || filterPriority"
              class="clear-filter"
              type="button"
              @click="clearFilters"
            >
              ล้างตัวกรอง
            </button>
          </div>
        </div>

        <div v-if="errorMessage" class="alert-card">
          <span>!</span>
          <div><strong>โหลดข้อมูลงานไม่ได้</strong><p>{{ errorMessage }}</p></div>
          <button type="button" @click="loadTasks">ลองอีกครั้ง</button>
        </div>

        <div v-if="loading" class="board-grid" aria-label="กำลังโหลด">
          <div v-for="column in columns" :key="column.key" class="board-column">
            <div class="column-header skeleton-header" />
            <div v-for="n in 2" :key="n" class="task-card skeleton-card" />
          </div>
        </div>

        <div v-else class="board-grid">
          <article v-for="column in columns" :key="column.key" class="board-column">
            <header class="column-header">
              <div>
                <span class="status-icon" :class="column.key">
                  <svg v-if="column.key === 'todo'" viewBox="0 0 24 24"><circle cx="12" cy="12" r="8" /></svg>
                  <svg v-else-if="column.key === 'inprogress'" viewBox="0 0 24 24"><path d="M12 4a8 8 0 1 1-5.66 2.34" /><path d="M4 4v5h5" /></svg>
                  <svg v-else viewBox="0 0 24 24"><path d="m5 12 4 4L19 6" /></svg>
                </span>
                <div>
                  <h3>{{ column.label }}</h3>
                  <p>{{ column.caption }}</p>
                </div>
              </div>
              <span class="column-count">{{ tasksByStatus[column.key].length }}</span>
            </header>

            <div class="card-stack">
              <button class="quick-add" type="button" @click="openCreate(column.key)">
                <span>+</span> เพิ่มงานใน {{ column.label }}
              </button>

              <TransitionGroup name="task-list">
                <div
                  v-for="task in tasksByStatus[column.key]"
                  :key="task.id"
                  class="task-card"
                  :class="`priority-${task.priority}`"
                >
                  <div class="card-spectrum" />
                  <div class="card-topline">
                    <span class="priority-pill" :class="task.priority">
                      <span /> {{ priorityLabel(task.priority) }}
                    </span>
                    <button class="icon-button" type="button" aria-label="แก้ไขงาน" @click="openEdit(task)">
                      <svg viewBox="0 0 24 24"><path d="m14 5 5 5M4 20l4.5-1 10-10a2.1 2.1 0 0 0-3-3l-10 10L4 20Z" /></svg>
                    </button>
                  </div>

                  <button class="task-title" type="button" @click="openEdit(task)">{{ task.title }}</button>
                  <p class="task-description">{{ task.description || 'ยังไม่มีรายละเอียดสำหรับงานนี้' }}</p>

                  <div class="task-footer">
                    <span class="task-date">
                      <svg viewBox="0 0 24 24"><path d="M7 3v3M17 3v3M4 9h16M5 5h14a1 1 0 0 1 1 1v14H4V6a1 1 0 0 1 1-1Z" /></svg>
                      {{ formatDate(task.updated_at || task.created_at) }}
                    </span>
                    <CustomSelect
                      :model-value="task.status"
                      :options="statusOptions"
                      compact
                      @update:model-value="changeStatus(task, $event)"
                    />
                  </div>
                </div>
              </TransitionGroup>

              <div v-if="tasksByStatus[column.key].length === 0" class="empty-column">
                <span class="empty-spark">✦</span>
                <strong>พื้นที่ยังว่าง</strong>
                <p>สร้างงานใหม่หรือปรับตัวกรองเพื่อดูงาน</p>
              </div>
            </div>
          </article>
        </div>
      </section>
    </main>

    <footer>
      <a class="brand footer-brand" href="#"><span class="brand-mark small">T</span><span>TaskBoard</span></a>
      <span>Move work forward, beautifully.</span>
      <span>© {{ new Date().getFullYear() }}</span>
    </footer>

    <Transition name="modal">
      <div v-if="isFormOpen" class="modal-backdrop" @mousedown.self="closeForm">
        <section class="modal-card" role="dialog" aria-modal="true" :aria-labelledby="editingTask ? 'edit-title' : 'create-title'">
          <div class="modal-accent" />
          <header class="modal-header">
            <div>
              <span class="section-kicker">{{ editingTask ? 'EDIT TASK' : 'NEW TASK' }}</span>
              <h2 :id="editingTask ? 'edit-title' : 'create-title'">
                {{ editingTask ? 'แก้ไขรายละเอียดงาน' : 'เพิ่มงานใหม่ให้บอร์ด' }}
              </h2>
            </div>
            <button class="close-button" type="button" aria-label="ปิด" @click="closeForm">×</button>
          </header>

          <form class="task-form" @submit.prevent="saveTask">
            <label class="form-field">
              <span>ชื่องาน <b>*</b></span>
              <input ref="titleInput" v-model="form.title" maxlength="200" placeholder="เช่น ออกแบบหน้า Dashboard" />
              <small>{{ form.title.length }}/200</small>
            </label>

            <label class="form-field">
              <span>รายละเอียด</span>
              <textarea v-model="form.description" rows="4" placeholder="บอกเป้าหมายหรือรายละเอียดที่ต้องทำ..." />
            </label>

            <div class="form-grid">
              <label class="form-field">
                <span>สถานะ</span>
                <CustomSelect v-model="form.status" :options="statusOptions" block />
              </label>
              <label class="form-field">
                <span>ความสำคัญ</span>
                <CustomSelect v-model="form.priority" :options="priorityOptions" block />
              </label>
            </div>

            <p v-if="formError" class="form-error">{{ formError }}</p>

            <div class="modal-actions">
              <button v-if="editingTask" class="danger-button" type="button" @click="requestDelete(editingTask)">
                <svg viewBox="0 0 24 24"><path d="M4 7h16M9 7V4h6v3M7 7l1 13h8l1-13M10 11v5M14 11v5" /></svg>
                ลบงาน
              </button>
              <span class="action-spacer" />
              <button class="secondary-button" type="button" @click="closeForm">ยกเลิก</button>
              <button class="primary-button save-button" type="submit" :disabled="saving">
                <span v-if="saving" class="spinner" />
                {{ saving ? 'กำลังบันทึก' : editingTask ? 'บันทึกการแก้ไข' : 'สร้างงาน' }}
              </button>
            </div>
          </form>
        </section>
      </div>
    </Transition>

    <Transition name="modal">
      <div v-if="taskToDelete" class="modal-backdrop danger-backdrop" @mousedown.self="taskToDelete = null">
        <section class="confirm-card" role="alertdialog" aria-modal="true">
          <span class="confirm-icon">
            <svg viewBox="0 0 24 24"><path d="M12 8v5M12 17h.01" /><path d="M10.3 3.7 2.6 17a2 2 0 0 0 1.7 3h15.4a2 2 0 0 0 1.7-3L13.7 3.7a2 2 0 0 0-3.4 0Z" /></svg>
          </span>
          <h2>ลบ “{{ taskToDelete.title }}”?</h2>
          <p>งานนี้จะถูกนำออกจากบอร์ดถาวร และไม่สามารถย้อนกลับได้</p>
          <div>
            <button class="secondary-button" type="button" @click="taskToDelete = null">เก็บงานไว้</button>
            <button class="danger-button solid" type="button" :disabled="saving" @click="deleteTask">
              {{ saving ? 'กำลังลบ' : 'ลบงานนี้' }}
            </button>
          </div>
        </section>
      </div>
    </Transition>

    <div class="toast-stack" aria-live="polite">
      <TransitionGroup name="toast">
        <div v-for="toast in toasts" :key="toast.id" class="toast" :class="toast.type">
          <span>{{ toast.type === 'success' ? '✓' : '!' }}</span>
          <p>{{ toast.message }}</p>
          <button type="button" @click="removeToast(toast.id)">×</button>
        </div>
      </TransitionGroup>
    </div>
  </div>
</template>

<script setup>
import { computed, nextTick, onMounted, reactive, ref } from 'vue'
import CustomSelect from './components/CustomSelect.vue'

const API = (import.meta.env.VITE_API_URL || '').replace(/\/$/, '')
const tasks = ref([])
const loading = ref(true)
const saving = ref(false)
const errorMessage = ref('')
const formError = ref('')
const searchQuery = ref('')
const filterPriority = ref('')
const isFormOpen = ref(false)
const editingTask = ref(null)
const taskToDelete = ref(null)
const titleInput = ref(null)
const boardSection = ref(null)
const toasts = ref([])
let toastId = 0

const form = reactive({ title: '', description: '', status: 'todo', priority: 'medium' })

const columns = [
  { key: 'todo', label: 'รอทำ', caption: 'งานที่รอเริ่มต้น' },
  { key: 'inprogress', label: 'กำลังทำ', caption: 'งานที่กำลังขับเคลื่อน' },
  { key: 'done', label: 'สำเร็จ', caption: 'งานที่ปิดเรียบร้อย' },
]

const statusOptions = [
  { value: 'todo', label: 'รอทำ', tone: 'neutral' },
  { value: 'inprogress', label: 'กำลังทำ', tone: 'violet' },
  { value: 'done', label: 'สำเร็จ', tone: 'green' },
]

const priorityOptions = [
  { value: 'low', label: 'ต่ำ', tone: 'green' },
  { value: 'medium', label: 'ปานกลาง', tone: 'amber' },
  { value: 'high', label: 'สูง', tone: 'rose' },
]

const priorityFilterOptions = [
  { value: '', label: 'ทุกความสำคัญ' },
  ...priorityOptions,
]

const particles = Array.from({ length: 18 }, (_, index) => ({
  id: index,
  style: {
    left: `${(index * 37) % 97}%`,
    top: `${8 + ((index * 53) % 86)}%`,
    animationDelay: `${(index % 7) * -0.8}s`,
    animationDuration: `${5 + (index % 5)}s`,
  },
}))

const counts = computed(() => ({
  todo: tasks.value.filter((task) => task.status === 'todo').length,
  inprogress: tasks.value.filter((task) => task.status === 'inprogress').length,
  done: tasks.value.filter((task) => task.status === 'done').length,
}))

const completionRate = computed(() => (
  tasks.value.length ? Math.round((counts.value.done / tasks.value.length) * 100) : 0
))

const filteredTasks = computed(() => {
  const query = searchQuery.value.trim().toLocaleLowerCase('th')
  return tasks.value.filter((task) => {
    const matchesPriority = !filterPriority.value || task.priority === filterPriority.value
    const matchesQuery = !query
      || task.title.toLocaleLowerCase('th').includes(query)
      || (task.description || '').toLocaleLowerCase('th').includes(query)
    return matchesPriority && matchesQuery
  })
})

const tasksByStatus = computed(() => Object.fromEntries(
  columns.map((column) => [
    column.key,
    filteredTasks.value.filter((task) => task.status === column.key),
  ]),
))

async function apiRequest(path, options = {}) {
  const response = await fetch(`${API}${path}`, {
    ...options,
    headers: {
      ...(options.body ? { 'Content-Type': 'application/json' } : {}),
      ...options.headers,
    },
  })

  let data = null
  try {
    data = await response.json()
  } catch {
    data = null
  }

  if (!response.ok) {
    throw new Error(data?.error || `คำขอล้มเหลว (${response.status})`)
  }
  return data
}

async function loadTasks() {
  loading.value = true
  errorMessage.value = ''
  try {
    tasks.value = await apiRequest('/api/tasks')
  } catch (error) {
    errorMessage.value = error.message || 'กรุณาตรวจสอบว่า Backend กำลังทำงาน'
  } finally {
    loading.value = false
  }
}

function resetForm(status = 'todo') {
  Object.assign(form, { title: '', description: '', status, priority: 'medium' })
  formError.value = ''
}

async function openCreate(status = 'todo') {
  editingTask.value = null
  resetForm(status)
  isFormOpen.value = true
  await nextTick()
  titleInput.value?.focus()
}

async function openEdit(task) {
  editingTask.value = task
  Object.assign(form, {
    title: task.title,
    description: task.description || '',
    status: task.status,
    priority: task.priority,
  })
  formError.value = ''
  isFormOpen.value = true
  await nextTick()
  titleInput.value?.focus()
}

function closeForm() {
  if (saving.value) return
  isFormOpen.value = false
  editingTask.value = null
}

async function saveTask() {
  if (!form.title.trim()) {
    formError.value = 'กรุณาใส่ชื่องานก่อนบันทึก'
    titleInput.value?.focus()
    return
  }

  saving.value = true
  formError.value = ''
  const payload = {
    title: form.title.trim(),
    description: form.description.trim(),
    status: form.status,
    priority: form.priority,
  }

  try {
    if (editingTask.value) {
      const updated = await apiRequest(`/api/tasks/${editingTask.value.id}`, {
        method: 'PUT',
        body: JSON.stringify(payload),
      })
      replaceTask(updated)
      pushToast('บันทึกการแก้ไขแล้ว')
    } else {
      const created = await apiRequest('/api/tasks', {
        method: 'POST',
        body: JSON.stringify(payload),
      })
      tasks.value.unshift(created)
      pushToast('สร้างงานใหม่เรียบร้อย')
    }
    closeForm()
  } catch (error) {
    formError.value = error.message
  } finally {
    saving.value = false
    if (!formError.value) {
      isFormOpen.value = false
      editingTask.value = null
    }
  }
}

async function changeStatus(task, status) {
  if (status === task.status) return
  const previousStatus = task.status
  task.status = status
  try {
    const updated = await apiRequest(`/api/tasks/${task.id}`, {
      method: 'PUT',
      body: JSON.stringify({ ...task, status }),
    })
    replaceTask(updated)
    pushToast(`ย้ายงานไป “${statusOptions.find((option) => option.value === status)?.label}” แล้ว`)
  } catch (error) {
    task.status = previousStatus
    pushToast(error.message, 'error')
  }
}

function requestDelete(task) {
  isFormOpen.value = false
  taskToDelete.value = task
}

async function deleteTask() {
  if (!taskToDelete.value) return
  saving.value = true
  try {
    await apiRequest(`/api/tasks/${taskToDelete.value.id}`, { method: 'DELETE' })
    tasks.value = tasks.value.filter((task) => task.id !== taskToDelete.value.id)
    pushToast('ลบงานออกจากบอร์ดแล้ว')
    taskToDelete.value = null
    editingTask.value = null
  } catch (error) {
    pushToast(error.message, 'error')
  } finally {
    saving.value = false
  }
}

function replaceTask(updated) {
  const index = tasks.value.findIndex((task) => task.id === updated.id)
  if (index !== -1) tasks.value[index] = updated
}

function priorityLabel(priority) {
  return priorityOptions.find((option) => option.value === priority)?.label || priority
}

function formatDate(date) {
  if (!date) return 'เพิ่งอัปเดต'
  return new Intl.DateTimeFormat('th-TH', { day: 'numeric', month: 'short' }).format(new Date(date))
}

function clearFilters() {
  searchQuery.value = ''
  filterPriority.value = ''
}

function scrollToBoard() {
  boardSection.value?.scrollIntoView({ behavior: 'smooth' })
}

function pushToast(message, type = 'success') {
  const id = ++toastId
  toasts.value.push({ id, message, type })
  window.setTimeout(() => removeToast(id), 3500)
}

function removeToast(id) {
  toasts.value = toasts.value.filter((toast) => toast.id !== id)
}

onMounted(loadTasks)
</script>
