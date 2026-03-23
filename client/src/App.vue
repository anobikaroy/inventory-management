<template>
  <div class="app-shell" :class="{ 'sidebar-collapsed': collapsed }">

    <!-- Mobile overlay -->
    <div v-if="mobileOpen" class="mobile-overlay" @click="mobileOpen = false"></div>

    <!-- LEFT SIDEBAR -->
    <aside class="sidebar" :class="{ open: mobileOpen }">
      <div class="sidebar-brand">
        <div class="brand-icon">C</div>
        <div v-show="!collapsed" class="brand-text">
          <h1 class="brand-name">{{ t('nav.companyName') }}</h1>
          <span class="brand-subtitle">{{ t('nav.subtitle') }}</span>
        </div>
      </div>

      <nav class="sidebar-nav">
        <router-link
          v-for="link in navLinks"
          :key="link.path"
          :to="link.path"
          class="nav-item"
          :class="{ active: $route.path === link.path }"
          :title="collapsed ? t(link.labelKey) : ''"
          @click="mobileOpen = false"
        >
          <span class="nav-icon">{{ link.icon }}</span>
          <span v-show="!collapsed" class="nav-label">{{ t(link.labelKey) }}</span>
        </router-link>
      </nav>

      <button class="collapse-toggle" @click="collapsed = !collapsed" :title="collapsed ? 'Expand sidebar' : 'Collapse sidebar'">
        <svg width="16" height="16" viewBox="0 0 16 16" fill="none">
          <path v-if="!collapsed" d="M10 4L6 8L10 12" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
          <path v-else d="M6 4L10 8L6 12" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
        </svg>
        <span v-show="!collapsed" class="collapse-label">Collapse</span>
      </button>

      <div class="sidebar-footer">
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </div>
    </aside>

    <!-- RIGHT BODY -->
    <div class="app-body">
      <div class="top-bar">
        <button class="hamburger" @click="mobileOpen = !mobileOpen" aria-label="Toggle navigation">
          <span></span>
          <span></span>
          <span></span>
        </button>
        <FilterBar />
      </div>
      <main class="main-content">
        <router-view />
      </main>
    </div>

    <ProfileDetailsModal
      :is-open="showProfileDetails"
      @close="showProfileDetails = false"
    />

    <TasksModal
      :is-open="showTasks"
      :tasks="tasks"
      @close="showTasks = false"
      @add-task="addTask"
      @delete-task="deleteTask"
      @toggle-task="toggleTask"
    />
  </div>
</template>

<script>
import { ref, onMounted, computed } from 'vue'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])
    const collapsed = ref(false)
    const mobileOpen = ref(false)

    const navLinks = [
      { path: '/',           labelKey: 'nav.overview',      icon: '◈' },
      { path: '/inventory',  labelKey: 'nav.inventory',     icon: '◻' },
      { path: '/orders',     labelKey: 'nav.orders',        icon: '◑' },
      { path: '/spending',   labelKey: 'nav.finance',       icon: '◈' },
      { path: '/demand',     labelKey: 'nav.demandForecast',icon: '◉' },
      { path: '/reports',    labelKey: 'nav.reports',       icon: '◧' },
      { path: '/restocking', labelKey: 'nav.restocking',    icon: '◐' },
    ]

    const tasks = computed(() => {
      return [...currentUser.value.tasks, ...apiTasks.value]
    })

    const loadTasks = async () => {
      try {
        apiTasks.value = await api.getTasks()
      } catch (err) {
        console.error('Failed to load tasks:', err)
      }
    }

    const addTask = async (taskData) => {
      try {
        const newTask = await api.createTask(taskData)
        apiTasks.value.unshift(newTask)
      } catch (err) {
        console.error('Failed to add task:', err)
      }
    }

    const deleteTask = async (taskId) => {
      try {
        const isMockTask = currentUser.value.tasks.some(t => t.id === taskId)
        if (isMockTask) {
          const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
          if (index !== -1) currentUser.value.tasks.splice(index, 1)
        } else {
          await api.deleteTask(taskId)
          apiTasks.value = apiTasks.value.filter(t => t.id !== taskId)
        }
      } catch (err) {
        console.error('Failed to delete task:', err)
      }
    }

    const toggleTask = async (taskId) => {
      try {
        const mockTask = currentUser.value.tasks.find(t => t.id === taskId)
        if (mockTask) {
          mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
        } else {
          const updatedTask = await api.toggleTask(taskId)
          const index = apiTasks.value.findIndex(t => t.id === taskId)
          if (index !== -1) apiTasks.value[index] = updatedTask
        }
      } catch (err) {
        console.error('Failed to toggle task:', err)
      }
    }

    onMounted(loadTasks)

    return {
      t,
      showProfileDetails,
      showTasks,
      tasks,
      collapsed,
      mobileOpen,
      navLinks,
      addTask,
      deleteTask,
      toggleTask
    }
  }
}
</script>

<style>
*, *::before, *::after {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

:root {
  --sidebar-width: 240px;
  --sidebar-collapsed-width: 64px;
  --color-bg: #f1f5f9;
  --color-surface: #ffffff;
  --color-border: #e2e8f0;
  --color-text-primary: #0f172a;
  --color-text-secondary: #64748b;
  --color-accent: #2563eb;
  --color-accent-light: #eff6ff;
  --sidebar-bg: #0f172a;
  --sidebar-border: #1e293b;
  --sidebar-text: #94a3b8;
  --sidebar-text-hover: #f1f5f9;
  --sidebar-text-active: #ffffff;
  --sidebar-hover-bg: #1e293b;
  --sidebar-active-bg: rgba(255, 255, 255, 0.08);
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  background: var(--color-bg);
  color: var(--color-text-primary);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* ── Shell ── */
.app-shell {
  display: flex;
  min-height: 100vh;
}

/* ── Sidebar ── */
.sidebar {
  width: var(--sidebar-width);
  flex-shrink: 0;
  background: var(--sidebar-bg);
  border-right: 1px solid var(--sidebar-border);
  display: flex;
  flex-direction: column;
  position: fixed;
  top: 0;
  left: 0;
  height: 100vh;
  z-index: 200;
  overflow-y: auto;
  overflow-x: hidden;
  transition: width 0.22s ease;
}

.app-shell.sidebar-collapsed .sidebar {
  width: var(--sidebar-collapsed-width);
}

.app-shell.sidebar-collapsed .app-body {
  margin-left: var(--sidebar-collapsed-width);
}

/* Brand */
.sidebar-brand {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 1.125rem 1rem;
  border-bottom: 1px solid var(--sidebar-border);
  min-height: 60px;
  overflow: hidden;
}

.brand-icon {
  width: 32px;
  height: 32px;
  border-radius: 8px;
  background: var(--color-accent);
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 800;
  font-size: 0.9375rem;
  flex-shrink: 0;
}

.brand-text {
  overflow: hidden;
  white-space: nowrap;
  min-width: 0;
}

.brand-name {
  font-size: 0.9375rem;
  font-weight: 700;
  color: #f8fafc;
  letter-spacing: -0.02em;
  line-height: 1.25;
}

.brand-subtitle {
  display: block;
  font-size: 0.6875rem;
  color: #475569;
  margin-top: 0.2rem;
  font-weight: 400;
}

/* Nav */
.sidebar-nav {
  flex: 1;
  padding: 0.625rem;
  display: flex;
  flex-direction: column;
  gap: 0.125rem;
  overflow: hidden;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.5625rem 0.75rem;
  border-radius: 7px;
  color: var(--sidebar-text);
  text-decoration: none;
  font-size: 0.875rem;
  font-weight: 500;
  transition: background 0.15s, color 0.15s;
  white-space: nowrap;
  overflow: hidden;
}

.nav-item:hover {
  background: var(--sidebar-hover-bg);
  color: var(--sidebar-text-hover);
}

.nav-item.active {
  background: var(--sidebar-active-bg);
  color: var(--sidebar-text-active);
  font-weight: 600;
}

.nav-icon {
  font-size: 1rem;
  width: 20px;
  text-align: center;
  flex-shrink: 0;
}

.nav-label {
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* Collapse toggle */
.collapse-toggle {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  margin: 0.5rem 0.625rem;
  padding: 0.5rem 0.75rem;
  background: none;
  border: 1px solid var(--sidebar-border);
  border-radius: 7px;
  color: var(--sidebar-text);
  cursor: pointer;
  font-size: 0.8125rem;
  font-weight: 500;
  font-family: inherit;
  transition: background 0.15s, color 0.15s;
  white-space: nowrap;
  overflow: hidden;
}

.collapse-toggle:hover {
  background: var(--sidebar-hover-bg);
  color: var(--sidebar-text-hover);
}

.collapse-label {
  font-size: 0.8125rem;
}

/* Sidebar footer */
.sidebar-footer {
  padding: 0.75rem 0.625rem;
  border-top: 1px solid var(--sidebar-border);
  display: flex;
  align-items: center;
  gap: 0.5rem;
  overflow: hidden;
  flex-shrink: 0;
}

/* ── App body ── */
.app-body {
  margin-left: var(--sidebar-width);
  flex: 1;
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  transition: margin-left 0.22s ease;
  min-width: 0;
}

/* ── Top bar ── */
.top-bar {
  position: sticky;
  top: 0;
  z-index: 90;
  background: var(--color-surface);
  border-bottom: 1px solid var(--color-border);
  display: flex;
  align-items: center;
}

.hamburger {
  display: none;
  flex-direction: column;
  gap: 5px;
  padding: 0.875rem 1rem;
  background: none;
  border: none;
  cursor: pointer;
  flex-shrink: 0;
}

.hamburger span {
  display: block;
  width: 20px;
  height: 2px;
  background: #64748b;
  border-radius: 2px;
}

/* ── Main content ── */
.main-content {
  flex: 1;
  padding: 1.75rem 2rem;
  max-width: 1400px;
  width: 100%;
}

/* ── Mobile overlay ── */
.mobile-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  z-index: 150;
}

/* ── Responsive ── */
@media (max-width: 768px) {
  .sidebar {
    transform: translateX(-100%);
    transition: transform 0.25s ease;
    width: var(--sidebar-width) !important;
  }

  .sidebar.open {
    transform: translateX(0);
  }

  .app-body {
    margin-left: 0 !important;
    transition: none;
  }

  .hamburger {
    display: flex;
  }
}

/* ────────────────────────────────────────
   Shared styles — all views depend on these
   ──────────────────────────────────────── */

.page-header {
  margin-bottom: 1.5rem;
}

.page-header h2 {
  font-size: 1.875rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.375rem;
  letter-spacing: -0.025em;
}

.page-header p {
  color: #64748b;
  font-size: 0.938rem;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1.25rem;
  margin-bottom: 1.5rem;
}

.stat-card {
  background: white;
  padding: 1.25rem;
  border-radius: 10px;
  border: 1px solid #e2e8f0;
  transition: all 0.2s ease;
}

.stat-card:hover {
  border-color: #cbd5e1;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.06);
}

.stat-label {
  color: #64748b;
  font-size: 0.875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 0.625rem;
}

.stat-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.stat-card.warning .stat-value { color: #ea580c; }
.stat-card.success .stat-value { color: #059669; }
.stat-card.danger .stat-value  { color: #dc2626; }
.stat-card.info .stat-value    { color: #2563eb; }
.stat-card.submitted .stat-value { color: #7c3aed; }

.card {
  background: white;
  border-radius: 10px;
  padding: 1.25rem;
  border: 1px solid #e2e8f0;
  margin-bottom: 1.25rem;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  padding-bottom: 0.875rem;
  border-bottom: 1px solid #e2e8f0;
}

.card-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: #f8fafc;
  border-top: 1px solid #e2e8f0;
  border-bottom: 1px solid #e2e8f0;
}

th {
  text-align: left;
  padding: 0.5rem 0.75rem;
  font-weight: 600;
  color: #475569;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

td {
  padding: 0.5rem 0.75rem;
  border-top: 1px solid #f1f5f9;
  color: #334155;
  font-size: 0.875rem;
}

tbody tr {
  transition: background-color 0.15s ease;
}

tbody tr:hover {
  background: #f8fafc;
}

.badge {
  display: inline-block;
  padding: 0.313rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.badge.success    { background: #d1fae5; color: #065f46; }
.badge.warning    { background: #fed7aa; color: #92400e; }
.badge.danger     { background: #fecaca; color: #991b1b; }
.badge.info       { background: #dbeafe; color: #1e40af; }
.badge.increasing { background: #d1fae5; color: #065f46; }
.badge.decreasing { background: #fecaca; color: #991b1b; }
.badge.stable     { background: #e0e7ff; color: #3730a3; }
.badge.submitted  { background: #ede9fe; color: #5b21b6; }
.badge.high       { background: #fecaca; color: #991b1b; }
.badge.medium     { background: #fed7aa; color: #92400e; }
.badge.low        { background: #dbeafe; color: #1e40af; }

.loading {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

.error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  font-size: 0.938rem;
}
</style>
