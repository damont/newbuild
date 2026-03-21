# Phase 05: Frontend Foundation

Build the React app skeleton with authentication, API client, routing, and the app shell. By the end of this phase, you can log in via the UI and see an authenticated layout.

**Prerequisite:** Phase 02 complete (backend auth endpoints working). Can run in parallel with Phases 03-04.

---

## Frontend Dependencies

Create `frontend/package.json`:

```json
{
  "name": "myapp-frontend",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^19.2.0",
    "react-dom": "^19.2.0"
  },
  "devDependencies": {
    "typescript": "~5.9.3",
    "vite": "^7.2.0",
    "@vitejs/plugin-react": "^5.1.0",
    "tailwindcss": "^4.1.0",
    "@tailwindcss/vite": "^4.1.0",
    "@types/react": "^19.2.0",
    "@types/react-dom": "^19.2.0"
  }
}
```

Install:

```bash
cd frontend && npm install
```

---

## Vite Configuration

Create `frontend/vite.config.ts`:

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [react(), tailwindcss()],
  server: {
    port: 8095,
    proxy: {
      '/api': {
        target: 'http://localhost:8020',
        changeOrigin: true,
      },
    },
  },
})
```

The proxy means the frontend can call `fetch('/api/things')` in dev without CORS issues. In production, nginx handles the same routing.

---

## TypeScript Config

Create `frontend/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedSideEffectImports": true
  },
  "include": ["src"]
}
```

---

## HTML Entry Point

Create `frontend/index.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>My App</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

---

## Vite Environment Types

Create `frontend/src/vite-env.d.ts`:

```typescript
/// <reference types="vite/client" />
```

This is required for TypeScript to recognize CSS imports and Vite-specific features.

---

## Styling

Create `frontend/src/index.css`:

```css
@import "tailwindcss";

:root {
  --bg-main: #111116;
  --bg-surface: #1a1a22;
  --bg-raised: #24242e;
  --border-color: #2e2e3a;
  --accent: #6c8aec;
  --accent-hover: #5a7ad4;
  --text-primary: #e4e4e8;
  --text-secondary: #9898a4;
  --text-muted: #5e5e6a;
  --selected-bg: rgba(108, 138, 236, 0.12);
  --header-bg: #14141a;
  --success: #4ade80;
  --warning: #facc15;
  --danger: #f87171;
}

body {
  margin: 0;
  min-height: 100vh;
  background-color: var(--bg-main);
  color: var(--text-primary);
}

#root {
  min-height: 100vh;
}
```

All components reference these CSS variables via Tailwind arbitrary values (e.g., `bg-[var(--bg-surface)]`) or inline styles.

---

## API Client

Create `frontend/src/api/client.ts`:

```typescript
class ApiClient {
  private baseUrl = ''

  getToken(): string | null {
    return localStorage.getItem('token')
  }

  setToken(token: string) {
    localStorage.setItem('token', token)
  }

  clearToken() {
    localStorage.removeItem('token')
  }

  isAuthenticated(): boolean {
    return this.getToken() !== null
  }

  private async request<T>(method: string, path: string, body?: unknown): Promise<T> {
    const headers: Record<string, string> = { 'Content-Type': 'application/json' }
    const token = this.getToken()
    if (token) {
      headers['Authorization'] = `Bearer ${token}`
    }

    const res = await fetch(`${this.baseUrl}${path}`, {
      method,
      headers,
      body: body ? JSON.stringify(body) : undefined,
    })

    if (res.status === 401) {
      this.clearToken()
      throw new Error('Unauthorized')
    }

    if (!res.ok) {
      throw new Error(`API error: ${res.status}`)
    }

    if (res.status === 204) return undefined as T
    return res.json()
  }

  get<T>(path: string) { return this.request<T>('GET', path) }
  post<T>(path: string, body: unknown) { return this.request<T>('POST', path, body) }
  put<T>(path: string, body: unknown) { return this.request<T>('PUT', path, body) }
  delete<T>(path: string) { return this.request<T>('DELETE', path) }
}

export const api = new ApiClient()
```

Key pattern: on a 401 response, clear the token and throw. The `AuthContext` detects the missing token on re-render and shows the login screen. Avoid `window.location.reload()` — it can cause infinite reload loops.

---

## Auth Context

Create `frontend/src/context/AuthContext.tsx`:

```typescript
import { createContext, useContext, useState, useEffect, ReactNode } from 'react'
import { api } from '../api/client'
import type { User } from '../types'

interface AuthContextType {
  isAuthenticated: boolean
  isLoading: boolean
  user: User | null
  login: (username: string, password: string) => Promise<void>
  logout: () => void
}

const AuthContext = createContext<AuthContextType | null>(null)

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null)
  const [isLoading, setIsLoading] = useState(true)

  useEffect(() => {
    if (api.isAuthenticated()) {
      api.get<User>('/api/auth/me')
        .then(setUser)
        .catch(() => api.clearToken())
        .finally(() => setIsLoading(false))
    } else {
      setIsLoading(false)
    }
  }, [])

  const login = async (username: string, password: string) => {
    const res = await api.post<{ access_token: string }>('/api/auth/login', { username, password })
    api.setToken(res.access_token)
    const userData = await api.get<User>('/api/auth/me')
    setUser(userData)
  }

  const logout = () => {
    api.clearToken()
    setUser(null)
  }

  return (
    <AuthContext.Provider value={{ isAuthenticated: !!user, isLoading, user, login, logout }}>
      {children}
    </AuthContext.Provider>
  )
}

export const useAuth = () => useContext(AuthContext)!
```

---

## Router Hook

Create `frontend/src/hooks/useRouter.ts`:

```typescript
import { useState, useEffect, useCallback } from 'react'

interface RouterState {
  path: string
  navigate: (path: string, replace?: boolean) => void
}

export function useRouter(): RouterState {
  const [path, setPath] = useState(() => window.location.pathname)

  useEffect(() => {
    const onPopState = () => setPath(window.location.pathname)
    window.addEventListener('popstate', onPopState)
    return () => window.removeEventListener('popstate', onPopState)
  }, [])

  const navigate = useCallback((to: string, replace = false) => {
    if (to === window.location.pathname) return
    if (replace) {
      window.history.replaceState({}, '', to)
    } else {
      window.history.pushState({}, '', to)
    }
    setPath(to)
  }, [])

  return { path, navigate }
}

export function matchPath(
  pattern: string,
  path: string,
): Record<string, string> | null {
  const patternParts = pattern.split('/').filter(Boolean)
  const pathParts = path.split('/').filter(Boolean)
  if (patternParts.length !== pathParts.length) return null
  const params: Record<string, string> = {}
  for (let i = 0; i < patternParts.length; i++) {
    if (patternParts[i].startsWith(':')) {
      params[patternParts[i].slice(1)] = pathParts[i]
    } else if (patternParts[i] !== pathParts[i]) {
      return null
    }
  }
  return params
}
```

---

## TypeScript Types

Create `frontend/src/types/index.ts`:

```typescript
export interface User {
  id: string
  username: string
  email: string
}

// Add more types here as resources are built.
// Keep these in sync with the backend DTOs.
```

---

## React Entry Point

Create `frontend/src/main.tsx`:

```typescript
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { AuthProvider } from './context/AuthContext'
import App from './App'
import './index.css'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <AuthProvider>
      <App />
    </AuthProvider>
  </StrictMode>,
)
```

---

## App Root

Create `frontend/src/App.tsx`:

```typescript
import { useEffect } from 'react'
import { useAuth } from './context/AuthContext'
import { useRouter } from './hooks/useRouter'

// Define your app's views here
type View = 'items' | 'settings'
const VIEWS: View[] = ['items', 'settings']

function pathToView(path: string): View | null {
  const first = path.split('/').filter(Boolean)[0]
  return VIEWS.includes(first as View) ? (first as View) : null
}

export default function App() {
  const { user, isLoading, login, logout } = useAuth()
  const { path, navigate } = useRouter()

  // Redirect authenticated users on / or unknown paths to default view
  useEffect(() => {
    if (!isLoading && user && !pathToView(path)) {
      navigate('/items', true)
    }
  }, [isLoading, user, path, navigate])

  // Redirect unauthenticated users on protected paths to /
  useEffect(() => {
    if (!isLoading && !user && path !== '/' && path !== '/login' && path !== '/register') {
      navigate('/', true)
    }
  }, [isLoading, user, path, navigate])

  if (isLoading) {
    return (
      <div className="min-h-screen flex items-center justify-center bg-[var(--bg-main)]">
        <p className="text-[var(--text-secondary)]">Loading...</p>
      </div>
    )
  }

  // Unauthenticated views
  if (!user) {
    if (path === '/register') {
      return <div>Register page (build in Phase 06)</div>
    }
    return <div>Login page (build in Phase 06)</div>
  }

  // Authenticated app shell
  const currentView = pathToView(path) || 'items'
  return (
    <div className="min-h-screen bg-[var(--bg-main)]">
      <header className="bg-[var(--header-bg)] border-b border-[var(--border-color)] px-4 py-3 flex items-center justify-between">
        <h1 className="text-lg font-semibold text-[var(--text-primary)]">My App</h1>
        <div className="flex items-center gap-4">
          <span className="text-sm text-[var(--text-secondary)]">{user.username}</span>
          <button
            onClick={logout}
            className="text-sm text-[var(--text-muted)] hover:text-[var(--text-primary)]"
          >
            Logout
          </button>
        </div>
      </header>
      <main className="p-4">
        <p className="text-[var(--text-secondary)]">Current view: {currentView}</p>
        <p className="text-sm text-[var(--text-muted)]">Build feature components in Phase 06</p>
      </main>
    </div>
  )
}
```

The key pattern: **derive the current view from the URL** rather than storing it in `useState`. The `navigate()` function updates the URL, which triggers a re-render, which reads the new view from the path. Back/forward, refresh, and deep-linking all work automatically.

---

## Verify

Start the frontend dev server (with the backend already running):

```bash
cd frontend && npm run dev
```

Visit `http://localhost:8095`. You should see:
1. A login/landing page when not authenticated
2. After logging in via the API (or building login UI), the authenticated app shell
3. URL changes when navigating between views

---

## Checklist

- [ ] `frontend/package.json` created, `npm install` succeeds
- [ ] `frontend/vite.config.ts` with proxy to backend port
- [ ] `frontend/tsconfig.json` configured
- [ ] `frontend/index.html` entry point
- [ ] `frontend/src/vite-env.d.ts` exists
- [ ] `frontend/src/index.css` with Tailwind import and theme variables
- [ ] `frontend/src/api/client.ts` — API client with token management
- [ ] `frontend/src/context/AuthContext.tsx` — auth state + login/logout
- [ ] `frontend/src/hooks/useRouter.ts` — History API wrapper + matchPath
- [ ] `frontend/src/types/index.ts` — User type (at minimum)
- [ ] `frontend/src/main.tsx` — React entry with AuthProvider
- [ ] `frontend/src/App.tsx` — auth gating + URL-driven routing shell
- [ ] `npm run dev` starts successfully
- [ ] App shows loading state, then login/authenticated shell
