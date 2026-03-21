# Phase 07: Frontend Testing

Set up Vitest with React Testing Library and write component tests.

**Prerequisite:** Phase 06 complete (feature components built).

---

## Setup

### Add Dependencies

Add to `frontend/package.json` devDependencies:

```json
"vitest": "^3.0.0",
"@testing-library/react": "^16.0.0",
"@testing-library/jest-dom": "^6.0.0",
"jsdom": "^25.0.0"
```

Add test scripts:

```json
"scripts": {
  "dev": "vite",
  "build": "tsc -b && vite build",
  "preview": "vite preview",
  "test": "vitest run",
  "test:watch": "vitest"
}
```

Install:

```bash
cd frontend && npm install
```

### Vite Config

Add the `test` block to `frontend/vite.config.ts`:

```typescript
export default defineConfig({
  plugins: [react(), tailwindcss()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: './src/test/setup.ts',
  },
  server: {
    // ...existing proxy config
  },
})
```

### Test Setup File

Create `frontend/src/test/setup.ts`:

```typescript
import '@testing-library/jest-dom'
```

---

## Test File Placement

Place test files next to the source files they test:

```
frontend/
├── src/
│   ├── api/
│   │   ├── client.ts
│   │   └── client.test.ts
│   ├── components/
│   │   └── tasks/
│   │       ├── TaskItem.tsx
│   │       └── TaskItem.test.tsx
│   └── test/
│       └── setup.ts
```

---

## What to Test

- **API client**: mock `fetch`, verify it sets auth headers, handles 401 correctly, clears tokens
- **Components with logic**: forms with validation, state-driven UI (expanded/collapsed, filtered lists)
- **Context providers**: auth login/logout flows update state correctly

### What to Skip

- Components that just render props with no logic
- Styling / visual appearance
- Third-party library behavior

---

## Example Tests

### API Client Test

```typescript
// frontend/src/api/client.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'

// Test the API client's behavior with mocked fetch
describe('ApiClient', () => {
  beforeEach(() => {
    localStorage.clear()
    vi.restoreAllMocks()
  })

  it('includes auth header when token exists', async () => {
    localStorage.setItem('token', 'test-token')
    const mockFetch = vi.fn().mockResolvedValue({
      ok: true,
      status: 200,
      json: () => Promise.resolve({ data: 'test' }),
    })
    vi.stubGlobal('fetch', mockFetch)

    const { api } = await import('./client')
    await api.get('/api/test')

    expect(mockFetch).toHaveBeenCalledWith(
      '/api/test',
      expect.objectContaining({
        headers: expect.objectContaining({
          Authorization: 'Bearer test-token',
        }),
      }),
    )
  })

  it('clears token on 401 response', async () => {
    localStorage.setItem('token', 'expired-token')
    vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
      ok: false,
      status: 401,
    }))

    const { api } = await import('./client')
    await expect(api.get('/api/test')).rejects.toThrow('Unauthorized')
    expect(localStorage.getItem('token')).toBeNull()
  })
})
```

### Component Test

```typescript
// frontend/src/components/tasks/TaskItem.test.tsx
import { describe, it, expect } from 'vitest'
import { render, screen } from '@testing-library/react'
import TaskItem from './TaskItem'

describe('TaskItem', () => {
  it('renders the task name', () => {
    render(<TaskItem name="Buy groceries" completed={false} />)
    expect(screen.getByText('Buy groceries')).toBeInTheDocument()
  })

  it('shows completed styling when completed', () => {
    render(<TaskItem name="Done task" completed={true} />)
    const element = screen.getByText('Done task')
    expect(element).toHaveClass('line-through')
  })
})
```

---

## Running Tests

```bash
cd frontend
npm test              # single run
npm run test:watch    # watch mode, re-runs on file changes
```

No backend or Docker needed — frontend tests mock all API calls.

---

## Checklist

- [ ] Vitest + React Testing Library + jsdom added to devDependencies
- [ ] `frontend/src/test/setup.ts` created
- [ ] `frontend/vite.config.ts` has `test` config block
- [ ] Test scripts added to `package.json` (`test`, `test:watch`)
- [ ] API client tests written (auth headers, 401 handling)
- [ ] Component tests written for components with non-trivial logic
- [ ] `npm test` passes all tests
