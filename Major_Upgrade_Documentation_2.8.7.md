# Major Upgrade Documentation
## react-query v3 → @tanstack/react-query v5 + React 18 → React 19

**Date:** 2026-06-12  
**Branches:** `upgrade/tanstack-query-v5` → `upgrade/react-19`  
**TypeScript errors before → after:** 331 → **0**  
**Files changed:** ~900+

---

## Table of Contents

1. [Overview](#overview)
2. [Package Changes](#package-changes)
3. [Phase 1 — @tanstack/react-query v5](#phase-1--tanstackreact-query-v5)
4. [Phase 2 — React 19](#phase-2--react-19)
5. [Phase 3 — Tooling & DX Improvements](#phase-3--tooling--dx-improvements)
6. [Breaking Change Reference](#breaking-change-reference)
7. [Developer Workflow Changes](#developer-workflow-changes)
8. [Known Limitations & Skipped Items](#known-limitations--skipped-items)

---

## Overview

This upgrade modernises the front-end stack across two major versions:

| Area | Before | After |
|---|---|---|
| React | 18.2.0 | **19.2.7** |
| React Query | react-query 3.x | **@tanstack/react-query 5.101** |
| Headless UI | @headlessui/react 1.7 | **2.2.10** |
| Stripe Elements | @stripe/react-stripe-js 1.x | **6.6.0** |
| Types | @types/react 18 | **@types/react 19** |

---

## Package Changes

### Removed
```
react-query                 (replaced by @tanstack/react-query)
react-loading-overlay       (unused — 0 usages in source)
@testing-library/react      (unused — 0 usages in source)
@testing-library/user-event (unused)
@testing-library/jest-dom   (unused)
```

### Added / Upgraded
```
@tanstack/react-query        3.x  →  5.101.0
react                       18.2.0 →  19.2.7
react-dom                   18.2.0 →  19.2.7
@types/react                18.x  →  19.x
@types/react-dom            18.x  →  19.x
@headlessui/react            1.7  →  2.2.10
@stripe/react-stripe-js      1.x  →  6.6.0
@stripe/stripe-js            1.x  →  9.8.0
react-compiler-runtime       —    →  1.0.0 (React Compiler runtime)
babel-plugin-react-compiler  —    →  1.0.0 (opt-in memoisation)
eslint-plugin-react-compiler —    →  19.1.0-rc.2 (lint support)
```

---

## Phase 1 — @tanstack/react-query v5

### What changed

#### 1. Central Compatibility Shim — `src/lib/reactQuery.ts`

All 686 source files import from `@/lib/reactQuery` — **never** directly from `@tanstack/react-query`. The shim provides the full v3 API surface over the v5 engine so existing call sites don't break.

```ts
// ✅ Always use this
import { useQuery, useMutation } from '@/lib/reactQuery'

// ❌ Never bypass the shim
import { useQuery } from '@tanstack/react-query'
```

The shim handles:
- All v3 positional overloads (`useQuery(key, fn, opts)`, `useQuery(key, opts)`)
- `onSuccess` / `onError` / `onSettled` callbacks via `useEffect`
- `keepPreviousData` → `placeholderData`
- `isLoading` alias for `isPending && fetchStatus !== 'idle'`
- `queryKey` normalisation — bare strings auto-wrapped in arrays

#### 2. Official Codemod Run

The v5 official codemod was run across the entire `src/` directory:

```
npx jscodeshift --transform @tanstack/react-query/.../remove-overloads.cjs
```

This converted ~800 files from positional to object form:
```ts
// BEFORE
useQuery(['key'], fetchFn, { staleTime: 1000 })

// AFTER
useQuery({ queryKey: ['key'], queryFn: fetchFn, staleTime: 1000 })
```

#### 3. String queryKeys → Array (Runtime Fix)

React Query v5 throws `"queryKey needs to be an Array"` at runtime for string keys. 5 files were fixed:

```ts
// ❌ BEFORE — runtime error
queryKey: 'GetItemCountInCartBySession' + sessionId + classId

// ✅ AFTER
queryKey: ['GetItemCountInCartBySession', sessionId, classId]
```

#### 4. Infrastructure — `useJustBatchCall.tsx` & `useJustCall.tsx`

Both files call `queryClient` methods directly (bypassing the shim), so they needed manual fixes:

```ts
// Added to both files
const toQueryKey = (key: string | string[]): string[] =>
  Array.isArray(key) ? key : [key]

// All queryClient calls updated:
queryClient.getQueryState(toQueryKey(q.key))
queryClient.prefetchQuery({ queryKey: toQueryKey(q.key), queryFn: q.fn })
queryClient.fetchQuery({ queryKey: toQueryKey(key) })   // v5 requires options object
```

Also fixed: `isFetching` → `fetchStatus === 'fetching'` (renamed field in v5 `QueryState`).

#### 5. `useGetPermission.ts` — Two Bugs Fixed

- Old 2-arg `useQuery(key, opts)` form → proper object form
- `validateResponse` returning `undefined` → added `return null` (v5 rejects undefined queryFn results)

### New Query Usage Patterns

```ts
// useQuery — basic
import { useQuery } from '@/lib/reactQuery'

const useGetMemberById = (memberId: string) => {
  return useQuery({
    queryKey: ['member', memberId],   // ✅ always an array
    queryFn: () => fetchMember(memberId),
    enabled: !!memberId,
    staleTime: 1000 * 60,
  })
}

// useMutation
const { mutate, isPending } = useMutation({
  mutationFn: (payload) => saveMember(payload),
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['member'] }),
})

// useInfiniteQuery — initialPageParam is now required
useInfiniteQuery({
  queryKey: ['items', filter],
  queryFn: ({ pageParam }) => fetchItems({ page: pageParam }),
  initialPageParam: 1,                          // ✅ required in v5
  getNextPageParam: (lastPage) => lastPage.nextPage,
})

// useQueries — must wrap in { queries: [...] }
useQueries({ queries: ids.map(id => ({ queryKey: ['member', id], queryFn: ... })) })
```

### Patterns to Avoid

```ts
// ❌ String queryKey
useQuery({ queryKey: 'my-key', queryFn: fn })

// ❌ onSuccess in useQuery options (removed in v5, shim emulates it)
useQuery({ queryKey: ['x'], queryFn: fn, onSuccess: (data) => {} })

// ❌ Old 2-arg form in new code (shim supports it but avoid writing new code this way)
useQuery('my-key', fetchFn, { staleTime: 1000 })

// ❌ queryClient.fetchQuery with bare key
queryClient.fetchQuery(key)          // v5 needs: queryClient.fetchQuery({ queryKey: key })
```

---

## Phase 2 — React 19

### What changed

#### 1. `useRef` Requires an Initializer

React 19 changed `useRef<T>()` (no argument) to require `useRef<T>(null)` or `useRef<T | null>(null)`.

```ts
// ❌ BEFORE — TypeScript error in React 19
const ref = useRef<HTMLDivElement>()

// ✅ AFTER
const ref = useRef<HTMLDivElement>(null)

// ✅ For mutable refs (zustand stores, timers etc.)
const storeRef = useRef<StoreApi<T>>(null)         // RefObject<T> — read-only current
const intervalRef = useRef<ReturnType<typeof setInterval>>(null)
```

**130+ files updated automatically.**

#### 2. Callback Refs Must Return `void`

Assignment expressions in `ref={}` callbacks now need to be wrapped in a block to return void:

```tsx
// ❌ BEFORE — returns value, TypeScript error
ref={(el) => (inputRefs.current[i] = el)}

// ✅ AFTER — returns void
ref={(el) => { inputRefs.current[i] = el }}
```

#### 3. `clearTimeout` / `clearInterval` with Nullable Refs

Timer refs typed as `NodeJS.Timeout | null` need null coalescing for `clearTimeout`:

```ts
// ❌ TypeScript error — clearTimeout doesn't accept null
clearTimeout(timerRef.current)

// ✅ Fix
clearTimeout(timerRef.current ?? undefined)
```

#### 4. JSX Namespace Removed from Global Scope

React 19 removed the global `JSX` namespace. Fixed by adding declarations to `src/react-app-env.d.ts`:

```ts
declare namespace JSX {
  interface Element extends React.JSX.Element {}
  interface IntrinsicElements extends React.JSX.IntrinsicElements {}
  // etc.
}
```

#### 5. SVG Module Type Override

Vite declares SVGs as `export default string`. Our SVGR plugin transforms them to React components. The type declaration was updated and `vite/client` moved to `tsconfig.json` `types[]` to prevent type merging conflicts:

```ts
// src/react-app-env.d.ts
declare module '*.svg' {
  const ReactComponent: React.FunctionComponent<React.SVGProps<SVGSVGElement> & { className?: string }>
  export { ReactComponent }
  export default ReactComponent
}
```

SVGs used as URL strings (e.g. `<img src={logo}>`) require a cast:
```tsx
<img src={logo as unknown as string} />
```

#### 6. `@headlessui/react` v1 → v2

**Breaking changes fixed:**
- `Dialog.Overlay` removed → replaced with plain `<div aria-hidden="true" />`
- `Transition` components needing `className`/`style` require `as="div"` in v2
- `Listbox.onChange` typed as `(value: unknown) => void` → cast at call sites
- Dot-notation sub-components deprecated (`Dialog.Panel` → `DialogPanel`) — **still works but will warn**

#### 7. React 19 `RefObject<T | null>` Compatibility

React 19's `useRef<T>(null)` returns `RefObject<T>` but with null internally. Internal hook signatures updated:

```ts
// ✅ Internal hooks now accept null
const useIsSticky = (nodeRef: React.RefObject<HTMLDivElement | null>) => ...
const useOnClickOutside = (ref: React.RefObject<HTMLElement | null>) => ...
```

---

## Phase 3 — Tooling & DX Improvements

### React Compiler (Opt-in Auto-Memoisation)

The React Compiler (`babel-plugin-react-compiler`) is enabled in **annotation mode** — only components/hooks with `'use memo'` are optimised.

```tsx
// Opt a component into the compiler
const MyComponent = () => {
  'use memo'
  // Compiler will auto-memoize this — no manual useMemo/useCallback needed
  const expensiveValue = computeSomething(props.data)
  return <div>{expensiveValue}</div>
}
```

**When NOT to use `'use memo'`:** Components with `try/catch` inside the render body, async/await in render, or disabled `react-hooks/*` ESLint rules.

#### Compiler Violations — VS Code Integration

Two layers of feedback:

1. **Dev server terminal** — shows which file/function was skipped:
   ```
   [React Compiler] ✗ Skipped
     File:   src/MyComponent.tsx:42:3
     Reason: Writing to a variable defined outside a component
   ```

2. **VS Code ESLint squiggles** — yellow underlines for:
   - Hooks rules violations (real correctness issues) via `eslint-plugin-react-compiler`
   - `try/catch` inside `'use memo'` components via custom `local/no-try-catch-in-use-memo` rule

#### Fix Patterns for Compiler Warnings

```tsx
// ❌ try/catch inside "use memo" — compiler skips entire component
const MyComp = () => {
  'use memo'
  const value = useMemo(() => {
    try { return JSON.parse(raw) } catch { return null }
  }, [raw])
}

// ✅ Move try/catch to a plain function outside the component
const safeParse = (raw: string) => {
  try { return JSON.parse(raw) } catch { return null }
}
const MyComp = () => {
  'use memo'
  const value = useMemo(() => safeParse(raw), [raw])
}
```

### Vite Config — `resolve.dedupe`

Added to prevent duplicate React instances when third-party packages bundle their own React:

```ts
resolve: {
  dedupe: ['react', 'react-dom', 'react/jsx-runtime'],
}
```

---

## Breaking Change Reference

### @tanstack/react-query v3 → v5

| v3 | v5 | Notes |
|---|---|---|
| `queryKey: 'string'` | `queryKey: ['string']` | Runtime error if string — use array |
| `useQuery(key, fn, opts)` | `useQuery({ queryKey, queryFn, ...opts })` | Shim handles old form |
| `onSuccess` in `useQuery` | `useEffect` on `isSuccess` | Removed from options in v5 |
| `status === 'loading'` | `status === 'pending'` | Status enum renamed |
| `isLoading` | `isPending && fetchStatus !== 'idle'` | Shim re-exposes `isLoading` |
| `keepPreviousData: true` | `placeholderData: (prev) => prev` | Shim maps automatically |
| `useQueries(array)` | `useQueries({ queries: array })` | Object wrapper required |
| `fetchInfiniteQuery` | requires `initialPageParam` | New required option |
| `QueryState.isFetching` | `QueryState.fetchStatus === 'fetching'` | Field restructured |

### React 18 → 19

| Pattern | Before | After | Notes |
|---|---|---|---|
| `useRef<T>()` | Works | Error | Must be `useRef<T>(null)` |
| `clearTimeout(ref.current)` | Works | Error | Must be `?? undefined` |
| Global `JSX` namespace | Available | Removed | Declared in `react-app-env.d.ts` |
| `RefObject<T>` | `T` only | `T \| null` internally | Hook signatures updated |
| `forwardRef` | Standard | Deprecated (warning) | Use ref-as-prop in new components |
| Callback ref returning value | Works | Error | Must return `void` |

### @headlessui/react v1 → v2

| v1 | v2 | Notes |
|---|---|---|
| `Dialog.Overlay` | Removed | Use plain `<div aria-hidden>` |
| `<Transition className>` | Error | Add `as="div"` prop |
| `Dialog.Panel`, `Tab.Group` etc. | Still works | Deprecated — migrate to `DialogPanel`, `TabGroup` etc. when convenient |
| `active` render prop | `focus` render prop | Renamed |
| `RadioGroup.Option` | `Radio` | Renamed |
| React peer dep | ≥16 | **≥18 required** |

---

## Developer Workflow Changes

### Adding New Queries

Always use the object form with array `queryKey`:

```ts
import { useQuery, useMutation, useQueryClient } from '@/lib/reactQuery'

export const useGetThing = (id: string) =>
  useQuery({
    queryKey: ['thing', id],        // ✅ array, specific
    queryFn: () => fetchThing(id),
    enabled: !!id,
  })
```

### Opting Into the React Compiler

Add `'use memo'` as the **first line** of the function body:

```tsx
const MyWidget = ({ data }: Props) => {
  'use memo'
  // Everything below is auto-memoized by the compiler
  const filtered = data.filter(isActive)
  return <List items={filtered} />
}
```

**Requirements to keep the compiler happy:**
- No `try/catch` inside the component body — extract to a module-level function
- No `async/await` in render
- No disabled `react-hooks/*` ESLint rules
- Follow all Rules of Hooks

### Checking for Compiler Issues

```bash
# See all files the compiler skips at dev-server startup
yarn dev   # watch for [React Compiler] ✗ Skipped messages

# See all ESLint violations (hooks rules + try/catch)
yarn lint  # filter for react-compiler and local/no-try-catch
```

---

## Known Limitations & Skipped Items

| Item | Status | Notes |
|---|---|---|
| `forwardRef` migration | ⏳ Deferred | 69 files still use `forwardRef`. Works in React 19 with deprecation warning. Migrate gradually to ref-as-prop |
| `@headlessui/react` dot-notation | ⏳ Deferred | `Dialog.Panel` etc. still work but show deprecation warnings. 44 files to update |
| `react-email-editor` | ⚠️ Abandoned library | 2 files use it. Works currently. Should be replaced with `@mailupinc/bee-plugin` (already installed) |
| `react-day-picker` v8 | ⚠️ Minor | v8 works with React 19. Upgrade to v9 when convenient for minor API improvements |
| `@justgo/planet-icons` React conflict | ✅ Fixed in library | Library was bundling its own React — fixed by externalising React in the library's build config |
| 7 `@ts-nocheck` files | ✅ Intentional | `justForm/forms/` demo files — pre-existing v2 `createJustForm` API mismatch, unrelated to this upgrade |

---

*Generated 2026-06-12 — for questions contact the team lead.*
