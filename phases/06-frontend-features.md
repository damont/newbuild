# Phase 06: Frontend Features

Build feature components, wire them into the URL-based routing system, and apply mobile-first responsive design.

**Prerequisite:** Phase 05 complete (React app shell with auth and routing working).

---

## Component Organization

Group components by feature, not by type:

```
components/
├── auth/           # Login.tsx, Register.tsx, LandingPage.tsx
├── layout/         # AppLayout.tsx, Header.tsx, Sidebar.tsx
├── tasks/          # TaskList.tsx, TaskItem.tsx, TaskDetail.tsx
└── categories/     # CategoryFilter.tsx, CategoryForm.tsx
```

Each component should:
- Be a single default export function component
- Accept props via a typed interface
- Use Tailwind utility classes directly — no separate CSS files
- Keep state local unless it needs to be shared (then lift to context)

---

## Auth Components

Build the auth pages referenced by `App.tsx`:

### Login.tsx

```typescript
// frontend/src/components/auth/Login.tsx
import { useState } from 'react'
import { useAuth } from '../../context/AuthContext'
import { useRouter } from '../../hooks/useRouter'

export default function Login() {
  const { login } = useAuth()
  const { navigate } = useRouter()
  const [username, setUsername] = useState('')
  const [password, setPassword] = useState('')
  const [error, setError] = useState('')

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setError('')
    try {
      await login(username, password)
      navigate('/items', true)
    } catch {
      setError('Invalid credentials')
    }
  }

  return (
    <div className="min-h-screen flex items-center justify-center bg-[var(--bg-main)] px-4">
      <form onSubmit={handleSubmit} className="w-full max-w-sm space-y-4">
        <h1 className="text-2xl font-bold text-[var(--text-primary)] text-center">Login</h1>
        {error && <p className="text-[var(--danger)] text-sm text-center">{error}</p>}
        <input
          type="text"
          placeholder="Username"
          value={username}
          onChange={e => setUsername(e.target.value)}
          className="w-full px-3 py-2 bg-[var(--bg-surface)] border border-[var(--border-color)] rounded text-[var(--text-primary)] min-h-[44px] sm:min-h-0"
        />
        <input
          type="password"
          placeholder="Password"
          value={password}
          onChange={e => setPassword(e.target.value)}
          className="w-full px-3 py-2 bg-[var(--bg-surface)] border border-[var(--border-color)] rounded text-[var(--text-primary)] min-h-[44px] sm:min-h-0"
        />
        <button
          type="submit"
          className="w-full py-2 bg-[var(--accent)] hover:bg-[var(--accent-hover)] text-white rounded font-medium min-h-[44px] sm:min-h-0"
        >
          Login
        </button>
        <p className="text-center text-sm text-[var(--text-secondary)]">
          Don't have an account?{' '}
          <a href="/register" onClick={e => { e.preventDefault(); navigate('/register') }} className="text-[var(--accent)]">
            Register
          </a>
        </p>
      </form>
    </div>
  )
}
```

Follow the same pattern for `Register.tsx` and `LandingPage.tsx`.

---

## Layout Components

### AppLayout.tsx

The authenticated app shell with header and optional sidebar/nav:

```typescript
// frontend/src/components/layout/AppLayout.tsx
import { useAuth } from '../../context/AuthContext'
import { useRouter } from '../../hooks/useRouter'

interface AppLayoutProps {
  currentView: string
  children: React.ReactNode
}

const NAV_ITEMS = [
  { key: 'items', label: 'Items' },
  { key: 'settings', label: 'Settings' },
]

export default function AppLayout({ currentView, children }: AppLayoutProps) {
  const { user, logout } = useAuth()
  const { navigate } = useRouter()

  return (
    <div className="min-h-screen bg-[var(--bg-main)]">
      <header className="bg-[var(--header-bg)] border-b border-[var(--border-color)] px-4 py-3">
        <div className="flex items-center justify-between">
          <h1 className="text-lg font-semibold text-[var(--text-primary)]">My App</h1>
          <div className="flex items-center gap-4">
            <span className="text-sm text-[var(--text-secondary)]">{user?.username}</span>
            <button
              onClick={logout}
              className="text-sm text-[var(--text-muted)] hover:text-[var(--text-primary)]"
            >
              Logout
            </button>
          </div>
        </div>
        <nav className="flex gap-1 mt-2 overflow-x-auto">
          {NAV_ITEMS.map(item => (
            <a
              key={item.key}
              href={`/${item.key}`}
              onClick={e => { e.preventDefault(); navigate(`/${item.key}`) }}
              className={`px-3 py-1.5 rounded text-sm min-h-[44px] sm:min-h-0 flex items-center ${
                currentView === item.key
                  ? 'bg-[var(--selected-bg)] text-[var(--accent)]'
                  : 'text-[var(--text-secondary)] hover:text-[var(--text-primary)]'
              }`}
            >
              {item.label}
            </a>
          ))}
        </nav>
      </header>
      <main className="p-4">{children}</main>
    </div>
  )
}
```

Key pattern for nav links: use `<a>` tags with `href` and `preventDefault`. This enables right-click "Open in new tab", middle-click, and URL on hover.

---

## URL Design

Plan URL paths for every navigable state:

| Path | What it shows |
|------|---------------|
| `/items` | List view |
| `/items/new` | Create form |
| `/items/:id` | Detail view |
| `/items/:id/edit` | Edit form |

---

## Deriving State from the URL

Replace `useState` for selection/mode with URL-derived values:

```typescript
// Before (broken back button, no deep links):
const [selectedId, setSelectedId] = useState<string | null>(null)

// After (URL is the source of truth):
const { path, navigate } = useRouter()
const selectedId = path.match(/^\/items\/(.+)/)?.[1] ?? null
// Click handler: navigate(`/items/${item.id}`)
// Close handler: navigate('/items')
```

State that is purely UI (search filters, sort mode, toggles) stays in `useState` — only navigable page transitions go in the URL.

### Using matchPath

For more complex routes, use the `matchPath` helper from `useRouter.ts`:

```typescript
import { matchPath } from '../hooks/useRouter'

const detailMatch = matchPath('/items/:id', path)
if (detailMatch) {
  const itemId = detailMatch.id
  // render detail view
}
```

---

## Mobile-First Design

### Principles

- **Start small**: Write the mobile layout as the default styles (no prefix). Add `sm:`, `md:`, `lg:` only when the layout needs to change at larger sizes.
- **Touch-friendly targets**: Interactive elements should be at least 44px tall on mobile. Use `min-h-[44px] sm:min-h-0`.
- **Stack on mobile, row on desktop**: Use `flex flex-col sm:flex-row`.
- **Full-width on mobile**: Buttons and inputs should be `w-full sm:w-auto`.
- **Responsive grids**: Use `grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-4` for card layouts.

### Common Patterns

```tsx
{/* Stack on mobile, row on desktop */}
<div className="flex flex-col sm:flex-row sm:items-center justify-between gap-3">
  <h2 className="text-lg font-semibold">Title</h2>
  <button className="w-full sm:w-auto min-h-[44px] sm:min-h-0 ...">Action</button>
</div>

{/* Responsive card grid */}
<div className="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-4 gap-3">
  {items.map(item => <Card key={item.id} ... />)}
</div>

{/* Horizontal scroll tabs on mobile */}
<nav className="flex gap-1 overflow-x-auto scrollbar-hide">
  {tabs.map(tab => <a ... />)}
</nav>
```

### Breakpoints (Tailwind defaults)

| Prefix | Min width | Typical device |
|--------|-----------|----------------|
| (none) | 0px | Phones |
| `sm:` | 640px | Large phones / small tablets |
| `md:` | 768px | Tablets |
| `lg:` | 1024px | Laptops / desktops |

Most apps only need `sm:` and `lg:`. Avoid overusing `md:` and `xl:`.

---

## Wiring Views into App.tsx

Update `App.tsx` to render actual components instead of placeholders:

```typescript
function ViewContent({ view }: { view: View }) {
  switch (view) {
    case 'items':
      return <ItemList />
    case 'settings':
      return <SettingsPage />
    default:
      return null
  }
}

// In the authenticated section of App:
return (
  <AppLayout currentView={currentView}>
    <ViewContent view={currentView} />
  </AppLayout>
)
```

---

## Checklist

- [ ] Auth components built (Login, Register, LandingPage)
- [ ] Layout components built (AppLayout with header and nav)
- [ ] Feature components organized by feature directory
- [ ] URL paths designed for all navigable views
- [ ] Views derive state from URL, not useState
- [ ] Nav links use `<a>` with `href` and `preventDefault`
- [ ] Mobile-first responsive design applied (touch targets, stacking, grids)
- [ ] All views reachable via URL navigation
- [ ] Browser back/forward works correctly
- [ ] Page refresh preserves the current view
