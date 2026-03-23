---
name: saas-ui-redesign
description: Redesigns a Vue 3 application's UI into a modern SaaS-style interface with a vertical navigation sidebar on the left, consistent spacing, and a polished professional look. Use this skill when asked to redesign, modernize, or switch to a sidebar/SaaS layout.
---

# SaaS UI Redesign Skill

This skill redesigns a Vue 3 application from a top horizontal navigation bar to a modern SaaS-style layout featuring a fixed vertical sidebar on the left, a top header strip, and a scrollable main content area.

---

## Phase 0 — Audit Before Touching Anything

Before writing a single line, read these files in full:

1. `client/src/App.vue` — current shell layout, global CSS variables, badge styles, stat card styles
2. `client/src/main.js` — all registered routes and their component imports
3. Every `.vue` file in `client/src/views/` — page-level `<style scoped>` blocks that reference `.page-header`, `.stats-grid`, `.card`, `.card-header`, etc.
4. `client/src/components/FilterBar.vue` — current sticky positioning (`top: 70px`)

Capture:
- The exact list of nav links (path + label)
- All CSS custom properties (e.g., `--primary`, color hex values already in use)
- Any component that imports or re-declares `.page-header`, `.stats-grid`, `.stat-card`, `.card` — these are global styles defined in `App.vue`'s non-scoped `<style>` block and must stay there

---

## Phase 1 — Layout Architecture

Replace the `<header class="top-nav">` + flat `<main class="main-content">` pattern with a two-column CSS Grid shell:

```
┌─────────────────────────────────────────────────┐
│  sidebar (fixed, full-height)  │  app-body      │
│  ┌───────────────────────────┐ │  ┌───────────┐ │
│  │  logo / brand             │ │  │ top-bar   │ │
│  │  ─────────────────────── │ │  │ (FilterBar│ │
│  │  nav links (vertical)     │ │  │  + header)│ │
│  │  ─────────────────────── │ │  ├───────────┤ │
│  │  bottom: profile / lang   │ │  │  main     │ │
│  └───────────────────────────┘ │  │  content  │ │
│                                 │  └───────────┘ │
└─────────────────────────────────────────────────┘
```

### App.vue template skeleton

```html
<template>
  <div class="app-shell">
    <!-- LEFT SIDEBAR -->
    <aside class="sidebar">
      <div class="sidebar-brand">
        <h1 class="brand-name">{{ t('nav.companyName') }}</h1>
        <span class="brand-subtitle">{{ t('nav.subtitle') }}</span>
      </div>

      <nav class="sidebar-nav">
        <router-link
          v-for="link in navLinks"
          :key="link.path"
          :to="link.path"
          class="nav-item"
          :class="{ active: $route.path === link.path }"
        >
          <span class="nav-icon">{{ link.icon }}</span>
          <span class="nav-label">{{ t(link.labelKey) }}</span>
        </router-link>
      </nav>

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
        <FilterBar />
      </div>
      <main class="main-content">
        <router-view />
      </main>
    </div>

    <!-- MODALS (unchanged) -->
    <ProfileDetailsModal ... />
    <TasksModal ... />
  </div>
</template>
```

Define `navLinks` as a computed or constant array in `setup()`:

```javascript
const navLinks = [
  { path: '/',           labelKey: 'nav.overview',       icon: '◈' },
  { path: '/inventory',  labelKey: 'nav.inventory',       icon: '◻' },
  { path: '/orders',     labelKey: 'nav.orders',          icon: '◑' },
  { path: '/spending',   labelKey: 'nav.finance',         icon: '◈' },
  { path: '/demand',     labelKey: 'nav.demandForecast',  icon: '◉' },
  { path: '/reports',    labelKey: 'nav.reports',         icon: '◧' },
  { path: '/restocking', labelKey: 'nav.restocking',      icon: '◐' },
]
```

> **No emoji** — use geometric Unicode symbols (◈ ◻ ◑ ◉ ◧ ◐) as neutral, professional icons consistent with the design system.

---

## Phase 2 — Global CSS Rewrite (App.vue non-scoped `<style>`)

Replace the entire existing `<style>` block with the following. Keep every rule that existing views depend on (`.badge.*`, `.stat-card.*`, `.card`, `.card-header`, `.table-container`, `table`, `th`, `td`, `.loading`, `.error`) — **only the shell layout rules change**.

### CSS variables & reset

```css
*, *::before, *::after { margin: 0; padding: 0; box-sizing: border-box; }

:root {
  --sidebar-width: 240px;
  --topbar-height: 56px;
  --color-bg: #f1f5f9;
  --color-surface: #ffffff;
  --color-border: #e2e8f0;
  --color-text-primary: #0f172a;
  --color-text-secondary: #64748b;
  --color-accent: #2563eb;
  --color-accent-light: #eff6ff;
  --radius-card: 10px;
  --shadow-card: 0 1px 3px 0 rgba(0,0,0,0.06), 0 1px 2px -1px rgba(0,0,0,0.04);
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  background: var(--color-bg);
  color: var(--color-text-primary);
  -webkit-font-smoothing: antialiased;
}
```

### Shell layout

```css
.app-shell {
  display: flex;
  min-height: 100vh;
}

/* ── Sidebar ── */
.sidebar {
  width: var(--sidebar-width);
  flex-shrink: 0;
  background: var(--color-surface);
  border-right: 1px solid var(--color-border);
  display: flex;
  flex-direction: column;
  position: fixed;
  top: 0;
  left: 0;
  height: 100vh;
  z-index: 100;
  overflow-y: auto;
}

.sidebar-brand {
  padding: 1.5rem 1.25rem 1.25rem;
  border-bottom: 1px solid var(--color-border);
}

.brand-name {
  font-size: 1rem;
  font-weight: 700;
  color: var(--color-text-primary);
  letter-spacing: -0.02em;
  line-height: 1.2;
}

.brand-subtitle {
  display: block;
  font-size: 0.7rem;
  color: var(--color-text-secondary);
  margin-top: 0.25rem;
  font-weight: 400;
}

.sidebar-nav {
  flex: 1;
  padding: 0.75rem 0.75rem;
  display: flex;
  flex-direction: column;
  gap: 0.125rem;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 0.625rem;
  padding: 0.5rem 0.75rem;
  border-radius: 7px;
  color: var(--color-text-secondary);
  text-decoration: none;
  font-size: 0.875rem;
  font-weight: 500;
  transition: background 0.15s, color 0.15s;
}

.nav-item:hover {
  background: var(--color-bg);
  color: var(--color-text-primary);
}

.nav-item.active {
  background: var(--color-accent-light);
  color: var(--color-accent);
  font-weight: 600;
}

.nav-icon {
  font-size: 1rem;
  width: 1.25rem;
  text-align: center;
  flex-shrink: 0;
}

.nav-label { flex: 1; }

.sidebar-footer {
  padding: 1rem 0.75rem;
  border-top: 1px solid var(--color-border);
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

/* ── App body ── */
.app-body {
  margin-left: var(--sidebar-width);
  flex: 1;
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

.top-bar {
  position: sticky;
  top: 0;
  z-index: 90;
  background: var(--color-surface);
  border-bottom: 1px solid var(--color-border);
}

.main-content {
  flex: 1;
  padding: 1.75rem 2rem;
  max-width: 1400px;
  width: 100%;
}
```

### FilterBar sticky fix

After the layout change, `FilterBar.vue`'s scoped style sets `top: 70px` (old nav height). Update it to `top: 0`:

```css
/* FilterBar.vue <style scoped> */
.filters-bar {
  top: 0; /* was 70px */
}
```

### Preserve all existing shared styles

Keep the following blocks **verbatim** from the old `App.vue` global style — views depend on them:

- `.page-header`, `.page-header h2`, `.page-header p`
- `.stats-grid`, `.stat-card`, `.stat-card:hover`, `.stat-label`, `.stat-value`
- `.stat-card.warning/success/danger/info .stat-value`
- `.card`, `.card-header`, `.card-title`
- `.table-container`, `table`, `thead`, `th`, `td`, `tbody tr`, `tbody tr:hover`
- `.badge` and all `.badge.*` variants (success, warning, danger, info, increasing, decreasing, stable, high, medium, low, submitted)
- `.loading`, `.error`

---

## Phase 3 — Component Adjustments

### LanguageSwitcher & ProfileMenu in sidebar footer

These components were originally in the top nav with `margin-right: 1rem`. In the sidebar footer they sit inside a flex row. Remove or override any `margin-left: auto` / `margin-right` rules that assumed horizontal nav placement. Verify visually or by reading each component's scoped styles.

### ProfileMenu dropdown direction

If `ProfileMenu.vue` opens a dropdown, its absolute positioning was anchored to the top-right corner of the screen. In the sidebar it sits at the bottom-left. Update the dropdown's `top`/`bottom` and `right`/`left` values so it opens upward (`bottom: 100%`) instead of downward.

---

## Phase 4 — Responsive Consideration (optional but recommended)

Add a simple mobile breakpoint so the sidebar collapses below 768px:

```css
@media (max-width: 768px) {
  .sidebar {
    transform: translateX(-100%);
    transition: transform 0.25s ease;
  }
  .sidebar.open {
    transform: translateX(0);
  }
  .app-body {
    margin-left: 0;
  }
}
```

Add a hamburger button in `.top-bar` that toggles a `.open` ref bound to the sidebar. Only implement this if the user has explicitly asked for mobile support.

---

## Phase 5 — Verification Checklist

After making all changes, verify each item before declaring done:

| # | Check | How to verify |
|---|-------|---------------|
| 1 | Sidebar renders on all pages | Navigate to every route |
| 2 | Active nav item highlighted | Each route shows correct active state |
| 3 | FilterBar is visible and sticky | Scroll down on `/orders`; bar should stick |
| 4 | No horizontal overflow | Sidebar + content should not exceed viewport width |
| 5 | Modals still open | Click profile avatar → Profile Details and Tasks |
| 6 | Badge colors intact | Check Orders page for Submitted/Delivered/etc. |
| 7 | Stats grid layout unchanged | Dashboard and Orders stat cards look correct |
| 8 | Language switcher works | Toggle EN/JA; all labels change |
| 9 | Dropdown opens upward | ProfileMenu dropdown does not go off-screen |
| 10 | No leftover `.top-nav` / `.nav-container` CSS | Search for orphaned selectors |

---

## Key Gotchas for This Codebase

1. **`top: 70px` in FilterBar.vue** — This magic number assumes the old nav height. Always update to `top: 0` when switching to sidebar layout.
2. **Global vs scoped styles** — `.stats-grid`, `.card`, `.badge` are in App.vue's *non-scoped* `<style>` block. Do not move them to scoped or to individual view files — every view depends on them.
3. **`nav-container > .nav-tabs`** — The old CSS had margin rules targeting `.nav-container > .nav-tabs` and `.nav-container > .language-switcher`. These become dead selectors after the redesign; remove them to avoid confusion.
4. **Fixed sidebar + scrollable body** — Set `position: fixed` on `.sidebar` and `margin-left: var(--sidebar-width)` on `.app-body`. Do NOT use `position: sticky` for the sidebar — it will scroll away.
5. **`ProfileMenu` dropdown z-index** — Ensure it is above the sidebar (`z-index > 100`) so it is not clipped.
6. **vue-expert agent** — Any modification to `.vue` files requires the `vue-expert` subagent per CLAUDE.md. Do not edit `.vue` files directly from the main context.
