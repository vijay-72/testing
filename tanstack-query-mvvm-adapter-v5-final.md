# TanStack Query MVVM Adapter: Complete Architecture Guide

## Document Purpose

This document provides a complete, production-ready architecture for integrating TanStack Query into MVVM-based applications (such as React Native with RxJS ViewModels). It explains not just *what* to build, but *why* each decision was made, drawing from lessons learned across four iterations of development.

---

## Table of Contents

1. [Problem Statement & Requirements](#1-problem-statement--requirements)
2. [Why TanStack Query?](#2-why-tanstack-query)
3. [Architecture Evolution: Lessons Learned](#3-architecture-evolution-lessons-learned)
4. [Core Concepts You Must Understand](#4-core-concepts-you-must-understand)
5. [Final Architecture Design](#5-final-architecture-design)
6. [Complete Implementation](#6-complete-implementation)
7. [Use Cases & Patterns](#7-use-cases--patterns)
8. [Testing Strategy](#8-testing-strategy)
9. [Platform Configuration](#9-platform-configuration)
10. [Migration Guide](#10-migration-guide)

---

## 1. Problem Statement & Requirements

### 1.1 The Problem We're Solving

In MVVM architecture, ViewModels manage application state and business logic while Views remain "dumb" presentational components. When it comes to server state (data fetched from APIs), we face several challenges:

**Without a solution like TanStack Query:**

```typescript
class VoicesViewModel extends BaseViewModel<VoicesState> {
  async fetchVoices() {
    // Problem 1: Manual loading state
    this.updateState({ isLoading: true, error: null })
    
    try {
      const voices = await VoiceAPI.getVoices()
      this.updateState({ voices, isLoading: false })
    } catch (error) {
      // Problem 2: Manual error handling
      this.updateState({ error: error.message, isLoading: false })
    }
    
    // Problem 3: No caching - every navigation refetches
    // Problem 4: No background refresh
    // Problem 5: No deduplication - two ViewModels = two requests
    // Problem 6: No stale-while-revalidate
    // Problem 7: No automatic refetch on network reconnect
    // Problem 8: No automatic refetch on app focus
    // Problem 9: No optimistic updates infrastructure
    // Problem 10: No request cancellation
  }
}
```

### 1.2 Requirements

| Requirement | Description | Priority |
|-------------|-------------|----------|
| **Caching** | Avoid redundant network requests across ViewModels | Critical |
| **Deduplication** | Multiple calls to same query = single request | Critical |
| **Background Refresh** | Update stale data without blocking UI | Critical |
| **Stale-While-Revalidate** | Show cached data while fetching fresh | Critical |
| **Loading/Error States** | Automatic state management | Critical |
| **Automatic Refetch** | On window focus, network reconnect | High |
| **Optimistic Updates** | Instant UI feedback for mutations | High |
| **Request Cancellation** | Cancel outdated requests | High |
| **Garbage Collection** | Automatic memory management | High |
| **TypeScript Support** | Full type inference | High |
| **Offline Support** | Persist cache across sessions | Medium |
| **Pagination** | Infinite scroll support | Medium |

### 1.3 Constraints

| Constraint | Implication |
|------------|-------------|
| **MVVM Architecture** | ViewModels own state, Views are dumb |
| **RxJS BehaviorSubject** | State flows through `state$` observable |
| **No React Hooks in VM** | Can't use `useQuery` directly |
| **ViewModel Lifecycle** | Must handle create/destroy properly |
| **Shared Cache** | Multiple ViewModels share same query cache |

---

## 2. Why TanStack Query?

### 2.1 What TanStack Query Provides

TanStack Query (formerly React Query) is a **framework-agnostic** async state management library. The core logic lives in `@tanstack/query-core`, while framework adapters provide idiomatic APIs.

**Key Insight:** React Query, Vue Query, Solid Query are all thin adapters on top of the same core. We can build our own MVVM adapter.

### 2.2 Core Capabilities

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      TanStack Query Capabilities                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  CACHING                      │  SYNCHRONIZATION                        │
│  ─────────                    │  ───────────────                        │
│  • In-memory cache            │  • Stale-while-revalidate               │
│  • Configurable staleTime     │  • Background refetching                │
│  • Configurable gcTime        │  • Refetch on window focus              │
│  • Query deduplication        │  • Refetch on network reconnect         │
│                               │  • Polling intervals                    │
│                                                                         │
│  MUTATIONS                    │  OPTIMIZATIONS                          │
│  ──────────                   │  ─────────────                          │
│  • Optimistic updates         │  • Request deduplication                │
│  • Rollback on error          │  • Automatic garbage collection         │
│  • Cache invalidation         │  • Structural sharing                   │
│  • Mutation queuing           │  • Property tracking                    │
│                                                                         │
│  DEVELOPER EXPERIENCE         │  PERSISTENCE (Optional)                 │
│  ────────────────────         │  ───────────────────────                │
│  • DevTools                   │  • localStorage                         │
│  • TypeScript inference       │  • IndexedDB                            │
│  • Query factories            │  • AsyncStorage (React Native)          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.3 The Hub-and-Spoke Architecture

```
                    ┌─────────────────────────────────────┐
                    │       @tanstack/query-core          │
                    │                                     │
                    │  QueryClient, QueryObserver,        │
                    │  QueryCache, MutationObserver,      │
                    │  focusManager, onlineManager        │
                    └──────────────┬──────────────────────┘
                                   │
         ┌─────────────┬───────────┼───────────┬─────────────┐
         │             │           │           │             │
         ▼             ▼           ▼           ▼             ▼
    ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
    │ React   │  │  Vue    │  │ Solid   │  │ Svelte  │  │  MVVM   │
    │ Adapter │  │ Adapter │  │ Adapter │  │ Adapter │  │ Adapter │
    │         │  │         │  │         │  │         │  │  (Ours) │
    │useQuery │  │useQuery │  │ create  │  │ create  │  │ this.   │
    │   ()    │  │   ()    │  │ Query() │  │ Query() │  │ query() │
    └─────────┘  └─────────┘  └─────────┘  └─────────┘  └─────────┘
```

---

## 3. Architecture Evolution: Lessons Learned

Understanding our mistakes is crucial to understanding why v4 is designed the way it is.

### 3.1 Version 1: Shared Observers (FAILED)

**What we tried:**
```typescript
class QueryCapability {
  // WRONG: Tried to reuse observers by query key
  private observers = new Map<string, QueryObserver>()
  private callbacks = new Map<string, Callback>()
  
  async fetch(options) {
    const hash = hashKey(options.queryKey)
    let observer = this.observers.get(hash)
    
    if (!observer) {
      observer = new QueryObserver(...)
      this.observers.set(hash, observer)
    }
    
    // Tried to swap callbacks when same query called again
    this.callbacks.set(hash, options.onStateChange)
  }
}
```

**Why it failed:**
1. **Closure Capture Bug**: The subscription callback captured the wrong `onStateChange`
2. **"First Callback Wins"**: Only the first caller's callback ever got called
3. **State Desynchronization**: Different ViewModels got out of sync

**The core misunderstanding:** We thought sharing observers would be "efficient." But deduplication happens at the **Query** level, not the Observer level. Observers are cheap - they just subscribe and notify.

### 3.2 Version 2: Better Callbacks, Missing Defaults

**What we tried:**
```typescript
// Better callback handling, but...
new QueryObserver(client, {
  queryKey: options.queryKey,
  queryFn: options.queryFn,
  staleTime: options.staleTime,
  // We cherry-picked options instead of using defaults
})
```

**Why it failed:**
- Missing `refetchOnWindowFocus`, `refetchOnReconnect`
- Missing `retry`, `retryDelay`
- Missing `networkMode`
- Missing query-specific defaults from `setQueryDefaults()`

**The core misunderstanding:** We thought we knew what options mattered. We didn't. TanStack Query has a sophisticated defaults system that merges global defaults, query-specific defaults, and provided options.

### 3.3 Version 3: QueryHandle Class (OVER-ENGINEERED)

**What we tried:**
```typescript
class QueryHandle {
  private observer: QueryObserver
  private currentHash: string
  
  // Manual hashing to detect changes
  private hashOptions(options): string {
    return JSON.stringify({
      queryKey: options.queryKey,
      enabled: options.enabled,
      staleTime: options.staleTime,
    })
  }
  
  checkAndUpdate(): void {
    const newHash = this.hashOptions(newOptions)
    if (newHash !== this.currentHash) {
      this.observer.setOptions(newOptions)
    }
  }
}
```

**Why it was wrong:**
- Added unnecessary abstraction layer
- Manual hashing was incomplete (what about `refetchInterval`? `select`?)
- Reimplemented what TanStack already does internally

**The core misunderstanding:** We thought we needed to optimize by checking if options changed. But both React and MobX just call `observer.setOptions()` every time and let TanStack handle it internally.

### 3.4 Version 4: Match React/MobX Exactly (CORRECT)

**The realization:** Stop being clever. Copy what works.

From React's `useBaseQuery`:
```typescript
// React Query - NO hashing, just call setOptions every render
React.useEffect(() => {
  observer.setOptions(defaultedOptions)
}, [defaultedOptions, observer])
```

From MobX-TanStack-Query:
```typescript
// MobX - NO hashing, just call setOptions
update(optionsUpdate) {
  this.options = { ...this.options, ...optionsUpdate }
  this.queryObserver.setOptions(this.options)
}
```

**v4 Design Principles:**
1. One observer per `query()` call (not shared)
2. Always use `defaultQueryOptions()` (get all defaults)
3. Just call `setOptions()` - TanStack handles comparison
4. No QueryHandle class, no custom hashing
5. Match the official adapters exactly

---

## 4. Core Concepts You Must Understand

### 4.1 The Query vs Observer Distinction

This is the most important concept:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│                         SHARED (In QueryCache)                          │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                        Query Instance                           │    │
│  │                     queryKey: ['voices']                        │    │
│  │                                                                 │    │
│  │  • data: Voice[]                                                │    │
│  │  • status: 'success'                                            │    │
│  │  • error: null                                                  │    │
│  │  • fetchStatus: 'idle'                                          │    │
│  │  • observers: [obs1, obs2, obs3]  ◄── Multiple!                 │    │
│  │                                                                 │    │
│  │  RESPONSIBILITIES:                                              │    │
│  │  • Execute fetch                                                │    │
│  │  • Deduplicate requests                                         │    │
│  │  • Manage retries                                               │    │
│  │  • Store data                                                   │    │
│  │  • Notify all observers                                         │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                      NOT SHARED (Per Component/VM)                      │
│                                                                         │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐              │
│  │  Observer 1 │      │  Observer 2 │      │  Observer 3 │              │
│  │ (VM-A)      │      │ (VM-B)      │      │ (VM-C)      │              │
│  │             │      │             │      │             │              │
│  │ callback-A  │      │ callback-B  │      │ callback-C  │              │
│  └─────────────┘      └─────────────┘      └─────────────┘              │
│                                                                         │
│  RESPONSIBILITIES:                                                      │
│  • Subscribe to Query                                                   │
│  • Call its own callback when Query changes                             │
│  • Track accessed properties (optimization)                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key Insight:**
- 3 ViewModels using `['voices']` = 3 Observers, 1 Query, 1 HTTP request
- Deduplication is automatic
- Each ViewModel gets its own notifications

### 4.2 The defaultQueryOptions() Function

**What it does:**
```typescript
const defaultedOptions = client.defaultQueryOptions(userOptions)
```

1. **Merges global defaults** from `new QueryClient({ defaultOptions: { queries: {...} } })`
2. **Merges query-specific defaults** from `setQueryDefaults(['voices'], {...})`
3. **Computes queryHash** for cache lookup
4. **Normalizes options** ensuring all required fields exist

**Why it's essential:**
```typescript
// Without defaultQueryOptions:
new QueryObserver(client, { queryKey: ['voices'], queryFn: fetch })
// Missing: staleTime, gcTime, retry, retryDelay, refetchOnWindowFocus,
// refetchOnReconnect, refetchOnMount, networkMode, structuralSharing, etc.

// With defaultQueryOptions:
const defaulted = client.defaultQueryOptions({ queryKey: ['voices'], queryFn: fetch })
new QueryObserver(client, defaulted)
// Gets ALL configured behaviors
```

### 4.3 getOptimisticResult() vs getCurrentResult()

```typescript
// getCurrentResult(): Returns what Query has RIGHT NOW
const result = observer.getCurrentResult()
// Problem: Observer just created, Query might not have updated yet

// getOptimisticResult(): Returns "best guess" based on options + cache
const result = observer.getOptimisticResult(defaultedOptions)
// Handles: placeholderData, initialData, keepPreviousData, isRestoring
```

**Example:**
```typescript
this.query({
  queryKey: ['user', id],
  queryFn: fetchUser,
  placeholderData: { name: 'Loading...' }  // Show while fetching
})

// getCurrentResult():
{ status: 'pending', data: undefined }  // Missing placeholder!

// getOptimisticResult():
{ status: 'success', data: { name: 'Loading...' }, isPlaceholderData: true }
```

### 4.4 The mount()/unmount() Lifecycle

```typescript
// What mount() does internally:
mount() {
  this.#mountCount++
  if (this.#mountCount === 1) {
    // Subscribe to focus manager
    this.#unsubscribeFocus = focusManager.subscribe(() => {
      if (focusManager.isFocused()) {
        this.#queryCache.onFocus()  // Triggers refetchOnWindowFocus!
      }
    })
    
    // Subscribe to online manager
    this.#unsubscribeOnline = onlineManager.subscribe(() => {
      if (onlineManager.isOnline()) {
        this.#queryCache.onOnline()  // Triggers refetchOnReconnect!
      }
    })
  }
}
```

**Without mount():**
- ❌ `refetchOnWindowFocus` doesn't work
- ❌ `refetchOnReconnect` doesn't work
- ❌ Paused mutations don't resume
- ❌ You lose major TanStack Query features

### 4.5 notifyManager.batchCalls()

```typescript
// Without batching:
// Multiple rapid updates → multiple callbacks → multiple state updates
Query changes → callback() → updateState()
Query changes → callback() → updateState()
Query changes → callback() → updateState()
// Result: 3 renders

// With batching:
observer.subscribe(notifyManager.batchCalls(callback))
// Multiple rapid updates → single batched callback
Query changes ─┐
Query changes ─┼─► batchCalls() → single callback()
Query changes ─┘
// Result: 1 render
```

---

## 5. Final Architecture Design

### 5.1 Mapping React Query to MVVM

| React Query | MVVM Adapter | Trigger |
|-------------|--------------|---------|
| `QueryClientProvider` | `initializeQueryClient()` | App startup |
| `useQuery()` in Component | `this.query()` in ViewModel | `initialize()` |
| Component re-render | `state$` change | `updateState()` |
| `observer.setOptions()` | `observer.setOptions()` | Same |
| Component unmount | `ViewModel.destroy()` | Screen unmount |
| `useMutation()` | `this.mutation()` | Same |

### 5.2 Data Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  INITIALIZATION FLOW                                                    │
│  ══════════════════                                                     │
│                                                                         │
│  1. App Startup                                                         │
│     └──► initializeQueryClient() → client.mount()                       │
│                                                                         │
│  2. Screen Mounts → ViewModel Created                                   │
│     └──► ViewModel.initialize()                                         │
│         └──► this.query(() => options)                                  │
│             └──► QueryCapability.fetch()                                │
│                 ├── getOptions() evaluates with current state           │
│                 ├── client.defaultQueryOptions() merges defaults        │
│                 ├── new QueryObserver() created                         │
│                 ├── getOptimisticResult() → onStateChange() ← immediate │
│                 ├── observer.subscribe() with batching                  │
│                 ├── observer.updateResult() catches races               │
│                 └── Entry stored: { getOptions, observer, unsubscribe } │
│                                                                         │
│  3. Query fetches data → observer notified → onStateChange()            │
│     └──► this.updateState({ data })                                     │
│         └──► state$.next() → UI renders                                 │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  DYNAMIC UPDATE FLOW (when state changes)                               │
│  ═══════════════════════════════════════                                │
│                                                                         │
│  1. User Action                                                         │
│     └──► this.updateState({ page: 2 })                                  │
│                                                                         │
│  2. BehaviorSubject.next()                                              │
│     └──► BaseViewModel's state$ subscription fires                      │
│         └──► queryCapability.onStateChange()                            │
│                                                                         │
│  3. For each query entry:                                               │
│     ├── getOptions() re-evaluates (now returns page: 2)                 │
│     ├── defaultQueryOptions() applies defaults                          │
│     └── observer.setOptions() → TanStack handles it                     │
│                                                                         │
│  4. TanStack internally:                                                │
│     ├── Compares old vs new options                                     │
│     ├── If queryKey changed → switches to different Query               │
│     ├── If enabled changed → starts/stops fetching                      │
│     └── Triggers fetch if needed → notifies our callback                │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  CLEANUP FLOW                                                           │
│  ════════════                                                           │
│                                                                         │
│  Screen Unmounts → ViewModel.destroy()                                  │
│  ├── queryCapability.destroy()                                          │
│  │   └── For each entry: unsubscribe()                                  │
│  └── state$.complete()                                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.3 File Structure

```
core/
├── query/
│   ├── index.ts              # Public exports
│   ├── queryClient.ts        # Singleton management (~50 lines)
│   ├── QueryCapability.ts    # Core adapter (~120 lines)
│   └── types.ts              # Type definitions (~60 lines)
│
├── viewmodels/
│   └── BaseViewModel.ts      # Integration (~25 lines added)
│
└── queries/                  # Query factories (per domain)
    ├── voiceQueries.ts
    ├── settingsQueries.ts
    └── userQueries.ts
```

---

## 6. Complete Implementation

### 6.1 types.ts

```typescript
import type {
  QueryKey,
  QueryObserverOptions,
  QueryObserverResult,
  MutationObserverOptions,
  MutationObserverResult,
  DefaultError,
} from '@tanstack/query-core'

/**
 * Query options extending TanStack's QueryObserverOptions.
 * 
 * The key addition is `onStateChange` - the bridge between
 * TanStack Query's observer pattern and our ViewModel's state$.
 */
export interface QueryOptions<
  TQueryFnData = unknown,
  TError = DefaultError,
  TData = TQueryFnData,
> extends Omit<
    QueryObserverOptions<TQueryFnData, TError, TData, TQueryFnData, QueryKey>,
    'queryKey'
  > {
  queryKey: QueryKey
  
  /**
   * Called on EVERY query state change:
   * 
   * - Immediately with optimistic/cached result (before fetch starts)
   * - When fetch starts (isFetching: true)
   * - When fetch completes (status: 'success' or 'error')
   * - On background refetch (existing data + isFetching: true)
   * - On invalidation from another ViewModel
   * - On window focus refetch (if refetchOnWindowFocus: true)
   * - On network reconnect refetch (if refetchOnReconnect: true)
   * - On polling interval (if refetchInterval is set)
   * 
   * This is the MVVM equivalent of React re-rendering.
   * Use it to update your ViewModel's state$.
   * 
   * @example
   * onStateChange: (result) => {
   *   this.updateState({
   *     voices: result.data ?? [],
   *     isLoading: result.isLoading,
   *     isFetching: result.isFetching,
   *     error: result.error?.message ?? null,
   *   })
   * }
   */
  onStateChange?: (result: QueryObserverResult<TData, TError>) => void
}

/**
 * Query options can be:
 * - Static object: Query doesn't depend on ViewModel state
 * - Function: Query re-evaluates when state changes (like React re-render)
 * 
 * @example
 * // Static - settings that don't change
 * this.query({
 *   queryKey: ['settings'],
 *   queryFn: fetchSettings,
 * })
 * 
 * // Dynamic - depends on current page
 * this.query(() => ({
 *   queryKey: ['voices', this.getStateValue().page],
 *   queryFn: () => fetchVoices(this.getStateValue().page),
 * }))
 */
export type QueryOptionsOrGetter<
  TQueryFnData = unknown,
  TError = DefaultError,
  TData = TQueryFnData,
> = 
  | QueryOptions<TQueryFnData, TError, TData>
  | (() => QueryOptions<TQueryFnData, TError, TData>)

/**
 * Mutation options extending TanStack's MutationObserverOptions.
 */
export interface MutationOptions<
  TData = unknown,
  TError = DefaultError,
  TVariables = void,
  TContext = unknown,
> extends MutationObserverOptions<TData, TError, TVariables, TContext> {
  /**
   * Called on EVERY mutation state change.
   * 
   * @example
   * onStateChange: (result) => {
   *   this.updateState({ isSaving: result.isPending })
   * }
   */
  onStateChange?: (
    result: MutationObserverResult<TData, TError, TVariables, TContext>
  ) => void
}

/**
 * Handle returned by this.mutation().
 * Provides imperative access to mutation functions.
 */
export interface MutationHandle<TData, TError, TVariables> {
  /** Fire-and-forget mutation */
  mutate: (variables: TVariables) => void
  
  /** Awaitable mutation */
  mutateAsync: (variables: TVariables) => Promise<TData>
  
  /** Get current mutation state */
  getState: () => MutationObserverResult<TData, TError, TVariables>
  
  /** Reset mutation to initial state */
  reset: () => void
}

// Re-export TanStack types for convenience
export type {
  QueryKey,
  QueryObserverResult,
  MutationObserverResult,
  DefaultError,
} from '@tanstack/query-core'

// Re-export queryOptions helper for type-safe query factories
export { queryOptions } from '@tanstack/query-core'
```

### 6.2 queryClient.ts

```typescript
import { QueryClient, type QueryClientConfig } from '@tanstack/query-core'

let queryClient: QueryClient | null = null

/**
 * Initialize the QueryClient singleton.
 * 
 * Call ONCE at app startup, before any ViewModel is created.
 * This is the MVVM equivalent of QueryClientProvider.
 * 
 * @example
 * // App.tsx or index.ts
 * initializeQueryClient({
 *   defaultOptions: {
 *     queries: {
 *       staleTime: 1000 * 60,      // 1 minute
 *       gcTime: 1000 * 60 * 5,     // 5 minutes
 *       retry: 3,
 *       refetchOnWindowFocus: true,
 *       refetchOnReconnect: true,
 *     }
 *   }
 * })
 */
export function initializeQueryClient(config?: QueryClientConfig): QueryClient {
  if (queryClient) {
    console.warn(
      'QueryClient already initialized. Returning existing instance. ' +
      'This is usually a bug - initializeQueryClient should only be called once.'
    )
    return queryClient
  }
  
  queryClient = new QueryClient(config)
  
  // CRITICAL: Mount the client to enable:
  // - refetchOnWindowFocus (via focusManager)
  // - refetchOnReconnect (via onlineManager)
  // - Paused mutation resumption
  // Without this, you lose major TanStack Query features!
  queryClient.mount()
  
  return queryClient
}

/**
 * Get the QueryClient singleton.
 * Throws if not initialized - catches bugs early.
 */
export function getQueryClient(): QueryClient {
  if (!queryClient) {
    throw new Error(
      'QueryClient not initialized. ' +
      'Call initializeQueryClient() at app startup, before any ViewModel is created.'
    )
  }
  return queryClient
}

/**
 * Clear all cached data.
 * 
 * Call on user logout to prevent data leaking between users.
 * Does NOT destroy the client - it can still be used.
 * 
 * @example
 * async function logout() {
 *   await AuthAPI.logout()
 *   clearQueryCache()
 *   navigation.reset({ routes: [{ name: 'Login' }] })
 * }
 */
export function clearQueryCache(): void {
  if (queryClient) {
    queryClient.clear()
  }
}

/**
 * Destroy the QueryClient completely.
 * 
 * Only call on app teardown or in tests.
 * After this, you must call initializeQueryClient() again.
 */
export function destroyQueryClient(): void {
  if (queryClient) {
    queryClient.unmount()
    queryClient = null
  }
}
```

### 6.3 QueryCapability.ts

```typescript
import {
  QueryObserver,
  MutationObserver,
  notifyManager,
  type QueryKey,
  type QueryObserverOptions,
  type QueryObserverResult,
  type MutationObserverResult,
} from '@tanstack/query-core'
import { getQueryClient } from './queryClient'
import type {
  QueryOptions,
  QueryOptionsOrGetter,
  MutationOptions,
  MutationHandle,
} from './types'

/**
 * Internal storage for a query.
 * Simple interface - no class needed.
 * 
 * WHY this design:
 * - getOptions: Function to re-evaluate options with current state
 * - observer: The TanStack QueryObserver instance
 * - unsubscribe: Cleanup function from observer.subscribe()
 */
interface QueryEntry {
  getOptions: () => QueryOptions<any, any, any>
  observer: QueryObserver<any, any, any, any, any>
  unsubscribe: () => void
}

/**
 * Internal storage for a mutation.
 */
interface MutationEntry {
  observer: MutationObserver<any, any, any, any>
  unsubscribe: () => void
}

/**
 * QueryCapability - The core adapter.
 * 
 * Each ViewModel instance gets its own QueryCapability instance.
 * This matches how each React component gets its own useQuery hooks.
 * 
 * KEY DESIGN DECISIONS:
 * 
 * 1. One observer per query() call (NOT shared)
 *    - Matches React Query behavior
 *    - Deduplication happens at Query level, not Observer level
 * 
 * 2. No custom hashing or comparison
 *    - Just call setOptions() and let TanStack handle it
 *    - This is exactly what React and MobX do
 * 
 * 3. No QueryHandle class
 *    - Simple QueryEntry interface is sufficient
 *    - Less code, same behavior
 */
export class QueryCapability {
  private queries: QueryEntry[] = []
  private mutations: MutationEntry[] = []
  private isDestroyed = false

  /**
   * Create and start a query.
   * 
   * This is the MVVM equivalent of useQuery().
   * 
   * @param optionsOrGetter - Static options object OR function returning options
   *   - Static: Query doesn't depend on ViewModel state
   *   - Function: Query re-evaluates when state$ changes
   * 
   * @returns Promise resolving to data (for imperative usage)
   * 
   * IMPLEMENTATION NOTES (matching useBaseQuery):
   * 1. Always use defaultQueryOptions() - never cherry-pick
   * 2. Set _optimisticResults = 'optimistic' for proper initial state
   * 3. Create NEW observer (not shared)
   * 4. Call getOptimisticResult() for immediate onStateChange
   * 5. Subscribe with notifyManager.batchCalls()
   * 6. Call updateResult() to catch race conditions
   */
  async fetch<TQueryFnData = unknown, TError = Error, TData = TQueryFnData>(
    optionsOrGetter: QueryOptionsOrGetter<TQueryFnData, TError, TData>
  ): Promise<TData | undefined> {
    this.ensureNotDestroyed()
    
    const client = getQueryClient()
    
    // Normalize to function (static options become a function that returns them)
    const getOptions = typeof optionsOrGetter === 'function'
      ? optionsOrGetter
      : () => optionsOrGetter
    
    // Get initial options
    const options = getOptions()
    
    // CRITICAL: Apply ALL defaults (staleTime, retry, refetchOnWindowFocus, etc.)
    // This is exactly what useBaseQuery does
    const defaultedOptions = client.defaultQueryOptions(
      options as QueryObserverOptions
    )
    
    // Enable optimistic results for proper handling of:
    // - placeholderData
    // - initialData
    // - keepPreviousData
    // - isRestoring state
    defaultedOptions._optimisticResults = 'optimistic'
    
    // Create NEW observer for THIS query() call
    // NOT shared - each call gets its own observer
    const observer = new QueryObserver(client, defaultedOptions)
    
    // Get optimistic result BEFORE subscribing
    // This handles placeholderData, initialData, etc.
    const optimisticResult = observer.getOptimisticResult(defaultedOptions)
    
    // Prime UI immediately with current/optimistic state
    if (options.onStateChange) {
      options.onStateChange(optimisticResult as QueryObserverResult<TData, TError>)
    }
    
    // Subscribe for future updates with batching
    // batchCalls() prevents multiple rapid updates from causing multiple callbacks
    const unsubscribe = observer.subscribe(
      notifyManager.batchCalls(() => {
        if (this.isDestroyed) return
        
        // Re-get options to get current onStateChange callback
        // (the callback closure might reference current state)
        const currentOptions = getOptions()
        currentOptions.onStateChange?.(
          observer.getCurrentResult() as QueryObserverResult<TData, TError>
        )
      })
    )
    
    // Catch updates between observer creation and subscription
    // This prevents race conditions where Query updated while we were setting up
    observer.updateResult()
    
    // Store entry for later cleanup and dynamic updates
    this.queries.push({ getOptions, observer, unsubscribe })
    
    // Return data for imperative usage
    const result = observer.getCurrentResult()
    
    // If we have fresh data, return immediately
    if (result.status === 'success' && !result.isStale) {
      return result.data as TData
    }
    
    // If disabled, just return current data (no fetch)
    if (defaultedOptions.enabled === false) {
      return result.data as TData | undefined
    }
    
    // Wait for fetch to complete
    try {
      const fetchResult = await observer.refetch()
      return fetchResult.data as TData | undefined
    } catch {
      // Error is already in observer state, onStateChange will have been called
      return observer.getCurrentResult().data as TData | undefined
    }
  }

  /**
   * Called by BaseViewModel when state$ changes.
   * 
   * Re-evaluates all query options and calls setOptions().
   * TanStack internally compares and decides what to do.
   * 
   * This is equivalent to React re-rendering and useQuery re-running.
   * 
   * IMPLEMENTATION NOTE:
   * We don't hash or compare options ourselves. We just call setOptions()
   * and let TanStack handle it. This is exactly what React and MobX do.
   */
  onStateChange(): void {
    if (this.isDestroyed) return
    
    const client = getQueryClient()
    
    for (const entry of this.queries) {
      // Re-evaluate options with current state
      const options = entry.getOptions()
      
      // Apply defaults (might have changed via setQueryDefaults)
      const defaultedOptions = client.defaultQueryOptions(
        options as QueryObserverOptions
      )
      defaultedOptions._optimisticResults = 'optimistic'
      
      // Just call setOptions - TanStack handles comparison internally
      // If queryKey changed → switches to different Query in cache
      // If enabled changed → starts/stops fetching
      // If staleTime changed → updates staleness calculation
      entry.observer.setOptions(defaultedOptions)
    }
  }

  /**
   * Create a mutation.
   * 
   * This is the MVVM equivalent of useMutation().
   */
  createMutation<
    TData = unknown,
    TError = Error,
    TVariables = void,
    TContext = unknown,
  >(
    options: MutationOptions<TData, TError, TVariables, TContext>
  ): MutationHandle<TData, TError, TVariables> {
    this.ensureNotDestroyed()
    
    const client = getQueryClient()
    const defaultedOptions = client.defaultMutationOptions(options)
    
    const observer = new MutationObserver(client, defaultedOptions)
    
    let unsubscribe = () => {}
    if (options.onStateChange) {
      unsubscribe = observer.subscribe(
        notifyManager.batchCalls(() => {
          if (!this.isDestroyed) {
            options.onStateChange!(
              observer.getCurrentResult() as MutationObserverResult<
                TData, TError, TVariables, TContext
              >
            )
          }
        })
      )
    }
    
    this.mutations.push({ observer, unsubscribe })
    
    return {
      mutate: (variables: TVariables) => observer.mutate(variables),
      mutateAsync: (variables: TVariables) => observer.mutateAsync(variables),
      getState: () => observer.getCurrentResult() as MutationObserverResult<
        TData, TError, TVariables
      >,
      reset: () => observer.reset(),
    }
  }

  /**
   * Prefetch data for future use.
   * 
   * WHY SEPARATE from query():
   * - NO observer created
   * - NO subscription
   * - NO onStateChange
   * Just warms the cache for later.
   * 
   * USE CASES:
   * - Hover prefetching (user might click)
   * - Route prefetching (user might navigate)
   * - Next page prefetching (infinite scroll)
   */
  async prefetch<TData = unknown>(options: {
    queryKey: QueryKey
    queryFn: () => Promise<TData>
    staleTime?: number
  }): Promise<void> {
    this.ensureNotDestroyed()
    await getQueryClient().prefetchQuery(options)
  }

  /**
   * Get cached data synchronously.
   * Returns undefined if not in cache.
   */
  getData<TData = unknown>(queryKey: QueryKey): TData | undefined {
    return getQueryClient().getQueryData(queryKey)
  }

  /**
   * Set query data directly.
   * 
   * USE CASES:
   * - Optimistic updates
   * - Updating cache after mutation
   * - Seeding cache with known data
   */
  setData<TData = unknown>(
    queryKey: QueryKey,
    updater: TData | ((old: TData | undefined) => TData)
  ): TData | undefined {
    return getQueryClient().setQueryData(queryKey, updater)
  }

  /**
   * Invalidate queries to trigger refetch.
   * 
   * USE CASES:
   * - After mutation success
   * - Forcing fresh data
   * - Cross-ViewModel cache invalidation
   */
  async invalidate(filters: {
    queryKey?: QueryKey
    exact?: boolean
    refetchType?: 'active' | 'inactive' | 'all' | 'none'
  } = {}): Promise<void> {
    await getQueryClient().invalidateQueries(filters)
  }

  /**
   * Cancel outgoing queries.
   * 
   * USE CASES:
   * - Before optimistic updates (prevent race conditions)
   * - Aborting slow requests
   */
  async cancel(filters: { queryKey?: QueryKey } = {}): Promise<void> {
    await getQueryClient().cancelQueries(filters)
  }

  /**
   * Clean up all observers and subscriptions.
   * 
   * MUST be called in ViewModel.destroy() to prevent:
   * - Memory leaks
   * - Callbacks to destroyed ViewModels
   * - Stale subscriptions
   */
  destroy(): void {
    this.isDestroyed = true
    
    for (const entry of this.queries) {
      entry.unsubscribe()
    }
    this.queries = []
    
    for (const entry of this.mutations) {
      entry.unsubscribe()
    }
    this.mutations = []
  }

  private ensureNotDestroyed(): void {
    if (this.isDestroyed) {
      throw new Error(
        'QueryCapability has been destroyed. ' +
        'This usually means you\'re calling query methods after ViewModel.destroy().'
      )
    }
  }
}
```

### 6.4 BaseViewModel.ts

```typescript
import { BehaviorSubject, type Subscription } from 'rxjs'
import { skip } from 'rxjs/operators'
import { QueryCapability } from '../query/QueryCapability'
import type {
  QueryKey,
  QueryOptionsOrGetter,
  MutationOptions,
  MutationHandle,
} from '../query/types'

/**
 * Base class for all ViewModels.
 * 
 * Provides:
 * - State management via BehaviorSubject
 * - TanStack Query integration via QueryCapability
 * - Automatic query re-evaluation on state changes
 */
export abstract class BaseViewModel<TState> {
  /** Observable state stream */
  protected state$: BehaviorSubject<TState>
  
  /** TanStack Query capability - one per ViewModel */
  private __queryCapability = new QueryCapability()
  
  /** Subscription to state$ for re-evaluating queries */
  private __stateSubscription: Subscription
  
  constructor(initialState: TState) {
    this.state$ = new BehaviorSubject(initialState)
    
    // When state changes, notify QueryCapability to re-evaluate queries
    // This is the MVVM equivalent of React component re-rendering
    // skip(1) ignores the initial value - we only care about changes
    this.__stateSubscription = this.state$.pipe(
      skip(1)
    ).subscribe(() => {
      this.__queryCapability.onStateChange()
    })
  }
  
  /** Get current state synchronously */
  protected getStateValue(): TState {
    return this.state$.getValue()
  }
  
  /** Update state with partial values */
  protected updateState(partial: Partial<TState>): void {
    this.state$.next({ ...this.state$.getValue(), ...partial })
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // Query API - Delegated to QueryCapability
  // ═══════════════════════════════════════════════════════════════════════════
  
  /**
   * Fetch data with TanStack Query.
   * This is the MVVM equivalent of useQuery().
   * 
   * @example
   * // Static query (doesn't depend on state)
   * await this.query({
   *   queryKey: ['settings'],
   *   queryFn: fetchSettings,
   *   onStateChange: (result) => this.updateState({ settings: result.data })
   * })
   * 
   * @example
   * // Dynamic query (re-evaluates when state changes)
   * await this.query(() => ({
   *   queryKey: ['voices', this.getStateValue().page],
   *   queryFn: () => fetchVoices(this.getStateValue().page),
   *   enabled: this.getStateValue().isReady,
   *   onStateChange: (result) => this.updateState({ voices: result.data })
   * }))
   */
  protected query<TQueryFnData = unknown, TError = Error, TData = TQueryFnData>(
    optionsOrGetter: QueryOptionsOrGetter<TQueryFnData, TError, TData>
  ): Promise<TData | undefined> {
    return this.__queryCapability.fetch(optionsOrGetter)
  }
  
  /**
   * Create a mutation.
   * This is the MVVM equivalent of useMutation().
   * 
   * @example
   * private saveMutation = this.mutation({
   *   mutationFn: (data: SaveData) => API.save(data),
   *   onSuccess: () => this.invalidateQueries({ queryKey: ['data'] }),
   *   onStateChange: (result) => this.updateState({ isSaving: result.isPending })
   * })
   * 
   * // Later:
   * this.saveMutation.mutate({ ... })
   */
  protected mutation<
    TData = unknown,
    TError = Error,
    TVariables = void,
    TContext = unknown,
  >(
    options: MutationOptions<TData, TError, TVariables, TContext>
  ): MutationHandle<TData, TError, TVariables> {
    return this.__queryCapability.createMutation(options)
  }
  
  /**
   * Prefetch data (cache warming, no observer).
   * 
   * @example
   * onItemHover(itemId: string) {
   *   this.prefetchQuery({
   *     queryKey: ['item', itemId],
   *     queryFn: () => fetchItem(itemId)
   *   })
   * }
   */
  protected prefetchQuery<TData = unknown>(options: {
    queryKey: QueryKey
    queryFn: () => Promise<TData>
    staleTime?: number
  }): Promise<void> {
    return this.__queryCapability.prefetch(options)
  }
  
  /** Get cached data synchronously */
  protected getQueryData<TData = unknown>(queryKey: QueryKey): TData | undefined {
    return this.__queryCapability.getData(queryKey)
  }
  
  /** Set query data directly (for optimistic updates) */
  protected setQueryData<TData = unknown>(
    queryKey: QueryKey,
    updater: TData | ((old: TData | undefined) => TData)
  ): TData | undefined {
    return this.__queryCapability.setData(queryKey, updater)
  }
  
  /** Invalidate queries to trigger refetch */
  protected invalidateQueries(filters?: {
    queryKey?: QueryKey
    exact?: boolean
    refetchType?: 'active' | 'inactive' | 'all' | 'none'
  }): Promise<void> {
    return this.__queryCapability.invalidate(filters)
  }
  
  /** Cancel outgoing queries */
  protected cancelQueries(filters?: { queryKey?: QueryKey }): Promise<void> {
    return this.__queryCapability.cancel(filters)
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // Lifecycle
  // ═══════════════════════════════════════════════════════════════════════════
  
  /**
   * Clean up ViewModel resources.
   * MUST be called when screen/component unmounts.
   */
  destroy(): void {
    this.__stateSubscription.unsubscribe()
    this.__queryCapability.destroy()
    this.state$.complete()
  }
}
```

### 6.5 index.ts (Public Exports)

```typescript
// Query client management
export {
  initializeQueryClient,
  getQueryClient,
  clearQueryCache,
  destroyQueryClient,
} from './queryClient'

// Types
export type {
  QueryOptions,
  QueryOptionsOrGetter,
  MutationOptions,
  MutationHandle,
  QueryKey,
  QueryObserverResult,
  MutationObserverResult,
} from './types'

// Re-export queryOptions for type-safe query factories
export { queryOptions } from '@tanstack/query-core'
```

---

## 7. Use Cases & Patterns

### 7.1 Basic Query

```typescript
interface VoicesState {
  voices: Voice[]
  isLoading: boolean
  error: string | null
}

class VoicesViewModel extends BaseViewModel<VoicesState> {
  constructor() {
    super({
      voices: [],
      isLoading: true,
      error: null,
    })
  }
  
  async initialize() {
    await this.query({
      queryKey: ['voices'],
      queryFn: VoiceAPI.getAll,
      staleTime: 5 * 60 * 1000, // 5 minutes
      
      onStateChange: (result) => {
        this.updateState({
          voices: result.data ?? [],
          isLoading: result.isLoading,
          error: result.error?.message ?? null,
        })
      }
    })
  }
}
```

### 7.2 Dynamic Query (Pagination)

```typescript
interface VoicesState {
  voices: Voice[]
  page: number
  search: string
  isFetching: boolean
}

class VoicesViewModel extends BaseViewModel<VoicesState> {
  constructor() {
    super({
      voices: [],
      page: 1,
      search: '',
      isFetching: false,
    })
  }
  
  async initialize() {
    // Options is a FUNCTION - re-evaluated when state changes
    await this.query(() => {
      const { page, search } = this.getStateValue()
      
      return {
        queryKey: ['voices', { page, search }],
        queryFn: () => VoiceAPI.getVoices({ page, search }),
        
        onStateChange: (result) => {
          this.updateState({
            voices: result.data ?? [],
            isFetching: result.isFetching,
          })
        }
      }
    })
  }
  
  setPage(page: number) {
    this.updateState({ page })
    // Query automatically re-evaluates with new page!
    // queryKey changes from ['voices', { page: 1, search: '' }]
    // to ['voices', { page: 2, search: '' }]
    // TanStack fetches page 2
  }
  
  setSearch(search: string) {
    this.updateState({ search, page: 1 }) // Reset to page 1 on search
    // Query automatically re-evaluates!
  }
}
```

### 7.3 Dependent Queries

```typescript
interface UserProfileState {
  userId: string | null
  user: User | null
  posts: Post[]
}

class UserProfileViewModel extends BaseViewModel<UserProfileState> {
  async initialize(userId: string) {
    this.updateState({ userId })
    
    // Query 1: Fetch user
    await this.query({
      queryKey: ['user', userId],
      queryFn: () => UserAPI.getUser(userId),
      onStateChange: (result) => {
        this.updateState({ user: result.data ?? null })
      }
    })
    
    // Query 2: Fetch posts - depends on user being loaded
    await this.query(() => {
      const { user } = this.getStateValue()
      
      return {
        queryKey: ['posts', user?.id],
        queryFn: () => PostAPI.getByUser(user!.id),
        enabled: !!user, // Only run when user exists
        onStateChange: (result) => {
          this.updateState({ posts: result.data ?? [] })
        }
      }
    })
  }
}
```

### 7.4 Mutation with Optimistic Update

```typescript
interface TodosState {
  todos: Todo[]
  isSaving: boolean
}

class TodosViewModel extends BaseViewModel<TodosState> {
  private addTodoMutation!: MutationHandle<Todo, Error, AddTodoInput>
  
  async initialize() {
    this.setupMutations()
    await this.fetchTodos()
  }
  
  private setupMutations() {
    this.addTodoMutation = this.mutation({
      mutationFn: (input: AddTodoInput) => TodoAPI.add(input),
      
      // Step 1: Before mutation, optimistically update UI
      onMutate: async (input) => {
        // Cancel any outgoing refetches
        await this.cancelQueries({ queryKey: ['todos'] })
        
        // Snapshot previous value (for rollback)
        const previousTodos = this.getQueryData<Todo[]>(['todos'])
        
        // Optimistically add the todo
        const optimisticTodo: Todo = {
          id: `temp-${Date.now()}`,
          text: input.text,
          completed: false,
        }
        
        this.setQueryData<Todo[]>(['todos'], (old) => 
          old ? [...old, optimisticTodo] : [optimisticTodo]
        )
        
        // Return context for rollback
        return { previousTodos }
      },
      
      // Step 2: On error, rollback
      onError: (error, input, context) => {
        if (context?.previousTodos) {
          this.setQueryData(['todos'], context.previousTodos)
        }
        this.updateState({ error: error.message })
      },
      
      // Step 3: On success or error, invalidate to ensure sync
      onSettled: () => {
        this.invalidateQueries({ queryKey: ['todos'] })
      },
      
      // Track mutation state
      onStateChange: (result) => {
        this.updateState({ isSaving: result.isPending })
      }
    })
  }
  
  addTodo(text: string) {
    this.addTodoMutation.mutate({ text })
  }
}
```

### 7.5 Prefetching

```typescript
class VoiceListViewModel extends BaseViewModel<ListState> {
  // Prefetch on hover - instant navigation
  onVoiceHover(voiceId: string) {
    this.prefetchQuery({
      queryKey: ['voice', voiceId],
      queryFn: () => VoiceAPI.getById(voiceId),
    })
  }
  
  // Prefetch next page when approaching end of list
  onApproachingEndOfList() {
    const nextPage = this.getStateValue().page + 1
    
    this.prefetchQuery({
      queryKey: ['voices', nextPage],
      queryFn: () => VoiceAPI.getVoices({ page: nextPage }),
    })
  }
}
```

### 7.6 Cross-ViewModel Cache Invalidation

```typescript
// ViewModel A: Manages list
class VoiceListViewModel extends BaseViewModel<ListState> {
  async initialize() {
    await this.query({
      queryKey: ['voices'],
      queryFn: VoiceAPI.getAll,
      onStateChange: (result) => {
        this.updateState({ voices: result.data ?? [] })
      }
    })
  }
}

// ViewModel B: Manages editing
class VoiceEditorViewModel extends BaseViewModel<EditorState> {
  private updateMutation!: MutationHandle<Voice, Error, UpdateInput>
  
  private setupMutations() {
    this.updateMutation = this.mutation({
      mutationFn: (input) => VoiceAPI.update(input.id, input.data),
      
      onSuccess: () => {
        // Invalidate ALL queries starting with 'voices'
        // This affects VoiceListViewModel even though it's a different instance!
        this.invalidateQueries({ queryKey: ['voices'] })
      }
    })
  }
}
```

### 7.7 Query Factory Pattern (Recommended)

```typescript
// queries/voiceQueries.ts
import { queryOptions } from '@tanstack/query-core'
import { VoiceAPI } from '../api/VoiceAPI'
import type { Voice, VoiceFilters } from '../types'

export const voiceQueries = {
  // Key-only entries (for invalidation)
  all: () => ['voices'] as const,
  lists: () => [...voiceQueries.all(), 'list'] as const,
  details: () => [...voiceQueries.all(), 'detail'] as const,
  
  // Full query options (for fetching)
  list: (filters?: VoiceFilters) =>
    queryOptions({
      queryKey: [...voiceQueries.lists(), filters] as const,
      queryFn: () => VoiceAPI.getVoices(filters),
      staleTime: 5 * 60 * 1000,
    }),
  
  detail: (id: string) =>
    queryOptions({
      queryKey: [...voiceQueries.details(), id] as const,
      queryFn: () => VoiceAPI.getVoice(id),
      staleTime: 10 * 60 * 1000,
    }),
}

// Usage in ViewModel:
class VoicesViewModel extends BaseViewModel<VoicesState> {
  async initialize() {
    await this.query({
      ...voiceQueries.list(),
      onStateChange: (result) => {
        this.updateState({ voices: result.data ?? [] })
      }
    })
  }
  
  async invalidateAll() {
    await this.invalidateQueries({ queryKey: voiceQueries.all() })
  }
}
```

---

## 8. Testing Strategy

### 8.1 Unit Testing QueryCapability

```typescript
import { QueryClient } from '@tanstack/query-core'
import { QueryCapability } from './QueryCapability'

// Mock getQueryClient
jest.mock('./queryClient', () => ({
  getQueryClient: jest.fn(),
}))

describe('QueryCapability', () => {
  let queryClient: QueryClient
  let capability: QueryCapability
  
  beforeEach(() => {
    queryClient = new QueryClient()
    queryClient.mount()
    require('./queryClient').getQueryClient.mockReturnValue(queryClient)
    capability = new QueryCapability()
  })
  
  afterEach(() => {
    capability.destroy()
    queryClient.unmount()
  })
  
  it('creates new observer for each fetch call', async () => {
    const onStateChange1 = jest.fn()
    const onStateChange2 = jest.fn()
    
    await capability.fetch({
      queryKey: ['test'],
      queryFn: () => Promise.resolve('data'),
      onStateChange: onStateChange1,
    })
    
    await capability.fetch({
      queryKey: ['test'], // Same key!
      queryFn: () => Promise.resolve('data'),
      onStateChange: onStateChange2,
    })
    
    // Both callbacks should be called - separate observers
    expect(onStateChange1).toHaveBeenCalled()
    expect(onStateChange2).toHaveBeenCalled()
  })
  
  it('calls onStateChange immediately with optimistic result', async () => {
    const onStateChange = jest.fn()
    
    // Start fetch but don't await
    capability.fetch({
      queryKey: ['test'],
      queryFn: () => new Promise((r) => setTimeout(() => r('data'), 100)),
      onStateChange,
    })
    
    // Should be called immediately with pending state
    expect(onStateChange).toHaveBeenCalledWith(
      expect.objectContaining({
        status: 'pending',
        isFetching: true,
      })
    )
  })
  
  it('re-evaluates dynamic queries on onStateChange', async () => {
    let currentPage = 1
    const getOptions = jest.fn(() => ({
      queryKey: ['voices', currentPage],
      queryFn: () => Promise.resolve([]),
      onStateChange: jest.fn(),
    }))
    
    await capability.fetch(getOptions)
    expect(getOptions).toHaveBeenCalledTimes(1)
    
    // Simulate state change
    currentPage = 2
    capability.onStateChange()
    
    // getOptions should be called again
    expect(getOptions).toHaveBeenCalledTimes(2)
  })
  
  it('stops receiving updates after destroy', async () => {
    const onStateChange = jest.fn()
    
    await capability.fetch({
      queryKey: ['test'],
      queryFn: () => Promise.resolve('data'),
      onStateChange,
    })
    
    const callCount = onStateChange.mock.calls.length
    
    capability.destroy()
    
    // Trigger cache update
    queryClient.setQueryData(['test'], 'new data')
    
    // Should not receive new calls
    expect(onStateChange.mock.calls.length).toBe(callCount)
  })
})
```

### 8.2 Integration Testing ViewModel

```typescript
import { initializeQueryClient, destroyQueryClient } from './queryClient'

describe('VoicesViewModel', () => {
  beforeEach(() => {
    initializeQueryClient()
  })
  
  afterEach(() => {
    destroyQueryClient()
  })
  
  it('fetches and updates state', async () => {
    const vm = new VoicesViewModel()
    const states: VoicesState[] = []
    
    vm.state$.subscribe((state) => states.push(state))
    
    await vm.initialize()
    
    // Should have loading state then success state
    expect(states.some((s) => s.isLoading === true)).toBe(true)
    expect(states[states.length - 1].voices.length).toBeGreaterThan(0)
    expect(states[states.length - 1].isLoading).toBe(false)
    
    vm.destroy()
  })
  
  it('receives background refetch updates', async () => {
    const vm = new VoicesViewModel()
    await vm.initialize()
    
    const onState = jest.fn()
    vm.state$.subscribe(onState)
    
    // Invalidate to trigger refetch
    await vm.invalidateQueries({ queryKey: ['voices'] })
    
    // Should have received update
    expect(onState).toHaveBeenCalled()
    
    vm.destroy()
  })
})
```

---

## 9. Platform Configuration

### 9.1 React Native Setup

```typescript
// app/querySetup.ts
import { AppState, type AppStateStatus } from 'react-native'
import NetInfo from '@react-native-community/netinfo'
import { focusManager, onlineManager } from '@tanstack/query-core'
import { initializeQueryClient } from './core/query/queryClient'

export function setupQueryClient() {
  // Configure focus manager for React Native
  // In RN, "focus" means app is in foreground
  focusManager.setEventListener((handleFocus) => {
    const subscription = AppState.addEventListener(
      'change',
      (state: AppStateStatus) => {
        handleFocus(state === 'active')
      }
    )
    return () => subscription.remove()
  })
  
  // Configure online manager for React Native
  onlineManager.setEventListener((setOnline) => {
    return NetInfo.addEventListener((state) => {
      setOnline(!!state.isConnected)
    })
  })
  
  // Initialize QueryClient
  initializeQueryClient({
    defaultOptions: {
      queries: {
        staleTime: 1000 * 60,        // 1 minute
        gcTime: 1000 * 60 * 5,       // 5 minutes
        retry: 3,
        retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
        refetchOnWindowFocus: true,   // Refetch when app returns to foreground
        refetchOnReconnect: true,     // Refetch when network reconnects
      },
      mutations: {
        retry: 1,
      },
    },
  })
}

// Call in App.tsx before rendering
setupQueryClient()
```

### 9.2 Web Setup

```typescript
// src/querySetup.ts
import { initializeQueryClient } from './core/query/queryClient'

export function setupQueryClient() {
  // focusManager and onlineManager work out of the box on web
  // They use window.addEventListener('focus') and navigator.onLine
  
  initializeQueryClient({
    defaultOptions: {
      queries: {
        staleTime: 1000 * 60,
        gcTime: 1000 * 60 * 5,
        retry: 3,
        refetchOnWindowFocus: true,
        refetchOnReconnect: true,
      },
    },
  })
}
```

---

## 10. Migration Guide

### 10.1 From Manual Fetching

**Before:**
```typescript
class VoicesViewModel extends BaseViewModel<VoicesState> {
  async fetchVoices() {
    this.updateState({ isLoading: true, error: null })
    
    try {
      const voices = await VoiceAPI.getVoices()
      this.updateState({ voices, isLoading: false })
    } catch (error) {
      this.updateState({ error: error.message, isLoading: false })
    }
  }
}
```

**After:**
```typescript
class VoicesViewModel extends BaseViewModel<VoicesState> {
  async initialize() {
    await this.query({
      queryKey: ['voices'],
      queryFn: VoiceAPI.getVoices,
      onStateChange: (result) => {
        this.updateState({
          voices: result.data ?? [],
          isLoading: result.isLoading,
          error: result.error?.message ?? null,
        })
      }
    })
  }
}
```

**Benefits gained:**
- ✅ Automatic caching
- ✅ Deduplication
- ✅ Background refetch
- ✅ Stale-while-revalidate
- ✅ Automatic retries
- ✅ Refetch on focus/reconnect

### 10.2 Checklist

- [ ] Install `@tanstack/query-core`
- [ ] Add QueryCapability to your project
- [ ] Update BaseViewModel
- [ ] Call `initializeQueryClient()` at app startup
- [ ] Configure platform-specific managers (React Native)
- [ ] Migrate ViewModels one at a time
- [ ] Create query factories for each domain
- [ ] Add `destroy()` calls to screen unmount handlers

---

## Summary

### Architecture Decisions and WHY

| Decision | Why |
|----------|-----|
| One observer per query() | Deduplication happens at Query level, not Observer level |
| Always use defaultQueryOptions() | Gets all configured behaviors (retry, refetchOnFocus, etc.) |
| No custom hashing | TanStack handles comparison internally |
| No QueryHandle class | Unnecessary abstraction - simple interface is sufficient |
| getOptimisticResult() for initial state | Handles placeholderData, initialData, etc. |
| notifyManager.batchCalls() | Prevents multiple rapid callbacks |
| mount() at app startup | Enables refetchOnWindowFocus, refetchOnReconnect |
| Function-based options | Enables dynamic queries that re-evaluate on state change |

### What We Get

| Feature | Implementation |
|---------|----------------|
| Caching | Via QueryCache (automatic) |
| Deduplication | Via Query (automatic) |
| Background Refresh | Via staleTime + refetch |
| Stale-While-Revalidate | Via QueryObserver |
| Loading/Error States | Via QueryObserverResult |
| Automatic Refetch | Via focusManager/onlineManager |
| Optimistic Updates | Via setQueryData() + mutations |
| TypeScript | Full inference via TanStack types |

### Code Stats

| File | Lines | Purpose |
|------|-------|---------|
| types.ts | ~80 | Type definitions |
| queryClient.ts | ~60 | Singleton management |
| QueryCapability.ts | ~200 | Core adapter |
| BaseViewModel.ts | +30 | Integration |
| **Total new code** | **~370** | Full TanStack Query integration |

---

*v5 Final - Complete architecture matching React Query exactly, with comprehensive WHY documentation*
