---
name: redesign-saas-ui
description: Redesigns a Vue 3 application's UI into a modern SaaS-style interface with a vertical navigation sidebar, consistent spacing, and a polished professional look.
---

# SaaS UI Redesign Skill

This skill transforms a Vue 3 application with a horizontal top navigation bar into a modern SaaS-style layout with a vertical sidebar, polished typography, and consistent spacing. You MUST delegate all `.vue` file creation and edits to the `vue-expert` agent.

## What This Skill Does

1. Replaces the horizontal top nav with a fixed vertical sidebar
2. Establishes a design token system (CSS custom properties)
3. Applies consistent spacing, typography, and color
4. Adds icon support to nav items
5. Moves secondary controls (profile, language) to the sidebar footer
6. Ensures the filter bar and main content area adapt to the new layout

---

## Step 0 — Audit Before Starting

Before writing any code, read these files to understand the current layout:

- `client/src/App.vue` — overall shell: nav, layout, global styles
- `client/src/main.js` — router and app bootstrap
- `client/src/api.js` — confirm API base URL / auth headers (no change needed)
- List every `<router-link>` in App.vue to inventory the nav items

Ask yourself:
- What components live inside the top nav (profile menu, language switcher, filter bar)?
- What CSS classes control the current layout (`.top-nav`, `.main-content`, etc.)?
- Are there global CSS custom properties already defined in App.vue?

---

## Step 1 — Design Token Update

In `App.vue`'s `<style>` (or a dedicated `variables.css`), replace or extend the `:root` block with these tokens. **Preserve any tokens already used by child components.**

```css
:root {
  /* Sidebar */
  --sidebar-width: 220px;
  --sidebar-bg: #0f172a;          /* slate-900 */
  --sidebar-border: #1e293b;      /* slate-800 */
  --sidebar-text: #94a3b8;        /* slate-400 */
  --sidebar-text-active: #f8fafc; /* slate-50  */
  --sidebar-accent: #3b82f6;      /* blue-500  */
  --sidebar-hover-bg: #1e293b;    /* slate-800 */
  --sidebar-active-bg: #1d4ed8;   /* blue-700  */

  /* Content area */
  --content-bg: #f8fafc;          /* slate-50  */
  --content-padding: 2rem;

  /* Cards */
  --card-bg: #ffffff;
  --card-border: #e2e8f0;         /* slate-200 */
  --card-radius: 0.75rem;
  --card-shadow: 0 1px 3px 0 rgb(0 0 0 / 0.1), 0 1px 2px -1px rgb(0 0 0 / 0.1);

  /* Typography */
  --font-sans: 'Inter', system-ui, -apple-system, sans-serif;
  --text-xs: 0.75rem;
  --text-sm: 0.875rem;
  --text-base: 1rem;
  --text-lg: 1.125rem;
  --text-xl: 1.25rem;
  --text-2xl: 1.5rem;

  /* Spacing scale */
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 0.75rem;
  --space-4: 1rem;
  --space-6: 1.5rem;
  --space-8: 2rem;

  /* Status colors (keep existing) */
  --color-success: #22c55e;
  --color-warning: #f59e0b;
  --color-danger:  #ef4444;
  --color-info:    #3b82f6;
}
```

---

## Step 2 — App Shell Restructure (App.vue)

Delegate to **vue-expert** with this specification.

### Template structure

Replace the existing `<div class="app">` shell with:

```html
<div class="app-shell">
  <!-- Sidebar -->
  <aside class="sidebar">
    <div class="sidebar-brand">
      <span class="sidebar-logo"><!-- SVG icon or initials --></span>
      <div class="sidebar-brand-text">
        <span class="sidebar-brand-name">{{ t('nav.companyName') }}</span>
        <span class="sidebar-brand-sub">{{ t('nav.subtitle') }}</span>
      </div>
    </div>

    <nav class="sidebar-nav">
      <router-link v-for="item in navItems" :key="item.to"
        :to="item.to"
        class="sidebar-link"
        :class="{ active: $route.path === item.to || $route.path.startsWith(item.to + '/') }"
      >
        <span class="sidebar-icon" v-html="item.icon"></span>
        <span class="sidebar-label">{{ item.label }}</span>
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

  <!-- Main area -->
  <div class="main-area">
    <FilterBar />
    <main class="main-content">
      <router-view />
    </main>
  </div>

  <!-- Modals (unchanged) -->
  <ProfileDetailsModal ... />
  <TasksModal ... />
</div>
```

### navItems data

Define in `setup()`:

```javascript
const { t } = useI18n()

const navItems = [
  {
    to: '/',
    label: computed(() => t('nav.overview')),
    icon: `<svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="7" height="7"/><rect x="14" y="3" width="7" height="7"/><rect x="3" y="14" width="7" height="7"/><rect x="14" y="14" width="7" height="7"/></svg>`
  },
  {
    to: '/inventory',
    label: computed(() => t('nav.inventory')),
    icon: `<svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 16V8a2 2 0 0 0-1-1.73l-7-4a2 2 0 0 0-2 0l-7 4A2 2 0 0 0 3 8v8a2 2 0 0 0 1 1.73l7 4a2 2 0 0 0 2 0l7-4A2 2 0 0 0 21 16z"/></svg>`
  },
  {
    to: '/orders',
    label: computed(() => t('nav.orders')),
    icon: `<svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M9 5H7a2 2 0 0 0-2 2v12a2 2 0 0 0 2 2h10a2 2 0 0 0 2-2V7a2 2 0 0 0-2-2h-2"/><rect x="9" y="3" width="6" height="4" rx="1"/></svg>`
  },
  {
    to: '/spending',
    label: computed(() => t('nav.finance')),
    icon: `<svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="12" y1="1" x2="12" y2="23"/><path d="M17 5H9.5a3.5 3.5 0 0 0 0 7h5a3.5 3.5 0 0 1 0 7H6"/></svg>`
  },
  {
    to: '/demand',
    label: computed(() => t('nav.demandForecast')),
    icon: `<svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="22 12 18 12 15 21 9 3 6 12 2 12"/></svg>`
  },
  {
    to: '/restocking',
    label: 'Restocking',
    icon: `<svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M23 4v6h-6"/><path d="M1 20v-6h6"/><path d="M3.51 9a9 9 0 0 1 14.85-3.36L23 10M1 14l4.64 4.36A9 9 0 0 0 20.49 15"/></svg>`
  },
  {
    to: '/reports',
    label: 'Reports',
    icon: `<svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/><line x1="16" y1="13" x2="8" y2="13"/><line x1="16" y1="17" x2="8" y2="17"/><polyline points="10 9 9 9 8 9"/></svg>`
  }
]
```

**Note on active matching for `/`**: use exact match (`$route.path === '/'`) for the overview link to prevent it highlighting on every page.

### Layout CSS

```css
.app-shell {
  display: flex;
  min-height: 100vh;
  background: var(--content-bg);
  font-family: var(--font-sans);
}

/* Sidebar */
.sidebar {
  position: fixed;
  top: 0;
  left: 0;
  bottom: 0;
  width: var(--sidebar-width);
  background: var(--sidebar-bg);
  border-right: 1px solid var(--sidebar-border);
  display: flex;
  flex-direction: column;
  z-index: 50;
  overflow-y: auto;
}

.sidebar-brand {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-6) var(--space-4);
  border-bottom: 1px solid var(--sidebar-border);
  min-height: 64px;
}

.sidebar-logo {
  width: 32px;
  height: 32px;
  background: var(--sidebar-accent);
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-weight: 700;
  font-size: var(--text-sm);
  flex-shrink: 0;
}

.sidebar-brand-text {
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.sidebar-brand-name {
  color: var(--sidebar-text-active);
  font-size: var(--text-sm);
  font-weight: 600;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.sidebar-brand-sub {
  color: var(--sidebar-text);
  font-size: var(--text-xs);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.sidebar-nav {
  flex: 1;
  padding: var(--space-4) var(--space-2);
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.sidebar-link {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-2) var(--space-3);
  border-radius: 6px;
  color: var(--sidebar-text);
  text-decoration: none;
  font-size: var(--text-sm);
  font-weight: 500;
  transition: background 0.15s, color 0.15s;
  white-space: nowrap;
}

.sidebar-link:hover {
  background: var(--sidebar-hover-bg);
  color: var(--sidebar-text-active);
}

.sidebar-link.active {
  background: var(--sidebar-active-bg);
  color: var(--sidebar-text-active);
}

.sidebar-icon {
  display: flex;
  align-items: center;
  flex-shrink: 0;
  opacity: 0.8;
}

.sidebar-link.active .sidebar-icon,
.sidebar-link:hover .sidebar-icon {
  opacity: 1;
}

.sidebar-footer {
  padding: var(--space-4);
  border-top: 1px solid var(--sidebar-border);
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
}

/* Main area */
.main-area {
  margin-left: var(--sidebar-width);
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.main-content {
  flex: 1;
  padding: var(--content-padding);
}
```

---

## Step 3 — FilterBar Placement

The `FilterBar` component moves from inside `<header>` to the top of `.main-area`, sitting above `<main>`. No changes to its internal logic. Update its style to fit the new context:

```css
/* FilterBar.vue — adjust top border/shadow to work as a sub-header */
.filter-bar {
  background: var(--card-bg);
  border-bottom: 1px solid var(--card-border);
  padding: var(--space-3) var(--content-padding);
  position: sticky;
  top: 0;
  z-index: 40;
}
```

---

## Step 4 — View-Level Polish

For each view in `client/src/views/`, apply these rules:

### Page header pattern
Every view should open with a consistent header block:

```html
<div class="page-header">
  <h1 class="page-title">Page Name</h1>
  <p class="page-subtitle">Optional description</p>
</div>
```

```css
.page-header {
  margin-bottom: var(--space-6);
}

.page-title {
  font-size: var(--text-2xl);
  font-weight: 700;
  color: #0f172a;
  margin: 0 0 var(--space-1);
  letter-spacing: -0.025em;
}

.page-subtitle {
  font-size: var(--text-sm);
  color: #64748b;
  margin: 0;
}
```

### Card pattern
All data panels, stat boxes, and tables should use the card token:

```css
.card {
  background: var(--card-bg);
  border: 1px solid var(--card-border);
  border-radius: var(--card-radius);
  box-shadow: var(--card-shadow);
  padding: var(--space-6);
}

.card-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: var(--space-4);
  padding-bottom: var(--space-4);
  border-bottom: 1px solid var(--card-border);
}

.card-title {
  font-size: var(--text-base);
  font-weight: 600;
  color: #0f172a;
  margin: 0;
}
```

### Stat tile pattern
For KPI/summary tiles (Dashboard, etc.):

```css
.stat-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: var(--space-4);
  margin-bottom: var(--space-6);
}

.stat-tile {
  background: var(--card-bg);
  border: 1px solid var(--card-border);
  border-radius: var(--card-radius);
  box-shadow: var(--card-shadow);
  padding: var(--space-6);
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
}

.stat-label {
  font-size: var(--text-xs);
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.stat-value {
  font-size: var(--text-2xl);
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.stat-change {
  font-size: var(--text-xs);
  font-weight: 500;
}

.stat-change.positive { color: var(--color-success); }
.stat-change.negative { color: var(--color-danger); }
```

---

## Step 5 — Table Polish

Apply to all `<table>` elements across views:

```css
.data-table {
  width: 100%;
  border-collapse: collapse;
  font-size: var(--text-sm);
}

.data-table th {
  text-align: left;
  padding: var(--space-3) var(--space-4);
  font-size: var(--text-xs);
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  border-bottom: 1px solid var(--card-border);
  background: #f8fafc;
}

.data-table td {
  padding: var(--space-3) var(--space-4);
  border-bottom: 1px solid #f1f5f9;
  color: #334155;
}

.data-table tr:last-child td {
  border-bottom: none;
}

.data-table tr:hover td {
  background: #f8fafc;
}
```

---

## Step 6 — Status Badge Pattern

Standardize status chips across Orders, Inventory, etc.:

```css
.badge {
  display: inline-flex;
  align-items: center;
  padding: 2px var(--space-2);
  border-radius: 9999px;
  font-size: var(--text-xs);
  font-weight: 600;
  text-transform: capitalize;
}

.badge-success  { background: #dcfce7; color: #166534; }
.badge-warning  { background: #fef3c7; color: #92400e; }
.badge-danger   { background: #fee2e2; color: #991b1b; }
.badge-info     { background: #dbeafe; color: #1e40af; }
.badge-neutral  { background: #f1f5f9; color: #475569; }
```

---

## Execution Order

Run these steps in sequence. Each step that touches a `.vue` file MUST go through the `vue-expert` agent.

1. Read current files (Step 0) — do this yourself
2. Update CSS tokens in `App.vue` — vue-expert
3. Restructure `App.vue` shell + sidebar + layout CSS — vue-expert
4. Adjust `FilterBar.vue` styles — vue-expert
5. Apply page-header + card + stat patterns to views, one view at a time — vue-expert per view
6. Apply table and badge patterns — vue-expert

After each step, use Playwright (`mcp__playwright__browser_take_screenshot`) to verify the layout at `http://localhost:3000`.

---

## Checklist

- [ ] Sidebar is fixed on the left, does not scroll with content
- [ ] Active nav item is visually distinct (highlighted background)
- [ ] Logo/brand visible at top of sidebar
- [ ] Profile and language controls are in the sidebar footer
- [ ] FilterBar is sticky at the top of the content area
- [ ] Content area has consistent `var(--content-padding)` on all sides
- [ ] All pages use the `.card` pattern for panels
- [ ] Tables use `.data-table` class with styled headers
- [ ] Status values use `.badge` with correct color class
- [ ] No horizontal scrollbar at 1280px viewport width
- [ ] App is free of console errors after redesign

---

## What NOT to Change

- API calls in `api.js` — layout changes don't affect data fetching
- Filter logic in composables — behavior stays the same
- Router configuration in `main.js`
- Pydantic models or backend code
- i18n translation keys — use existing `t()` calls
