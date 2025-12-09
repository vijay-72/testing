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
7. [Query Options & Factories](#7-query-options--factories)
8. [Use Cases & Patterns](#8-use-cases--patterns)
9. [Data Flow Diagrams](#9-data-flow-diagrams)
10. [Edge Cases & Error Handling](#10-edge-cases--error-handling)
11. [Testing Strategy](#11-testing-strategy)
12. [Platform Configuration](#12-platform-configuration)
13. [Migration Guide](#13-migration-guide)

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
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      TanStack Query Capabilities                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  CACHING                         │  SYNCHRONIZATION                             │
│  ────────                        │  ───────────────                             │
│  • In-memory cache               │  • Stale-while-revalidate                    │
│  • Configurable staleTime        │  • Background refetching                     │
│  • Configurable gcTime           │  • Refetch on window focus                   │
│  • Query deduplication           │  • Refetch on network reconnect              │
│                                  │  • Polling intervals                         │
│                                                                                 │
│  MUTATIONS                       │  OPTIMIZATIONS                               │
│  ─────────                       │  ─────────────                               │
│  • Optimistic updates            │  • Request deduplication                     │
│  • Rollback on error             │  • Automatic garbage collection              │
│  • Cache invalidation            │  • Structural sharing                        │
│  • Mutation queuing              │  • Property tracking                         │
│                                                                                 │
│  DEVELOPER EXPERIENCE            │  PERSISTENCE (Optional)                      │
│  ────────────────────            │  ───────────────────────                     │
│  • DevTools                      │  • localStorage                              │
│  • TypeScript inference          │  • IndexedDB                                 │
│  • Query factories               │  • AsyncStorage (React Native)               │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 The Hub-and-Spoke Architecture

```
                    ┌─────────────────────────────────────────┐
                    │       @tanstack/query-core              │
                    │                                         │
                    │  QueryClient, QueryObserver,            │
                    │  QueryCache, MutationObserver,          │
                    │  focusManager, onlineManager            │
                    └──────────────────┬──────────────────────┘
                                       │
         ┌─────────────────┬───────────┼───────────┬─────────────────┐
         │                 │           │           │                 │
         ▼                 ▼           ▼           ▼                 ▼
    ┌─────────┐       ┌─────────┐ ┌─────────┐ ┌─────────┐       ┌─────────┐
    │  React  │       │   Vue   │ │  Solid  │ │ Svelte  │       │  MVVM   │
    │ Adapter │       │ Adapter │ │ Adapter │ │ Adapter │       │ Adapter │
    │         │       │         │ │         │ │         │       │ (Ours)  │
    │useQuery │       │useQuery │ │ create  │ │ create  │       │ this.   │
    │   ()    │       │   ()    │ │ Query() │ │ Query() │       │ query() │
    └─────────┘       └─────────┘ └─────────┘ └─────────┘       └─────────┘
```

---

## 3. Architecture Evolution: Lessons Learned

Understanding our mistakes is crucial to understanding why v4/v5 is designed the way it is.

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

### 3.4 Version 4/5: Match React/MobX Exactly (CORRECT)

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

**v4/v5 Design Principles:**
1. One observer per `query()` call (not shared)
2. Always use `defaultQueryOptions()` (get all defaults)
3. Just call `setOptions()` - TanStack handles comparison
4. No QueryHandle class, no custom hashing
5. Match the official adapters exactly

---

## 4. Core Concepts You Must Understand

### 4.1 The Query vs Observer Distinction

This is the most important concept to understand:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  QueryCache (Singleton - lives for app lifetime)                                │
│  ════════════════════════════════════════════════                               │
│                                                                                 │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                     Query Instance ['voices']                             │  │
│  │                          (SHARED)                                         │  │
│  │                                                                           │  │
│  │  • data: Voice[]                                                          │  │
│  │  • status: 'success' | 'pending' | 'error'                                │  │
│  │  • error: Error | null                                                    │  │
│  │  • fetchStatus: 'fetching' | 'idle' | 'paused'                            │  │
│  │  • dataUpdatedAt: number                                                  │  │
│  │                                                                           │  │
│  │  RESPONSIBILITIES:                                                        │  │
│  │  ─────────────────                                                        │  │
│  │  • Execute queryFn (fetch data)                                           │  │
│  │  • Deduplicate concurrent requests                                        │  │
│  │  • Manage retries on failure                                              │  │
│  │  • Store data and metadata                                                │  │
│  │  • Track staleness (staleTime)                                            │  │
│  │  • Notify ALL attached observers                                          │  │
│  │                                                                           │  │
│  │  observers: [                                                             │  │
│  │    QueryObserver (VoicesListVM),                                          │  │
│  │    QueryObserver (VoiceDetailVM),                                         │  │
│  │    QueryObserver (SettingsVM)                                             │  │
│  │  ]                                                                        │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  QueryObserver Instances (NOT shared - one per query() call)                    │
│  ═══════════════════════════════════════════════════════════                    │
│                                                                                 │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐      │
│  │  QueryObserver 1    │  │  QueryObserver 2    │  │  QueryObserver 3    │      │
│  │  (VoicesListVM)     │  │  (VoiceDetailVM)    │  │  (SettingsVM)       │      │
│  │                     │  │                     │  │                     │      │
│  │  onStateChange:     │  │  onStateChange:     │  │  onStateChange:     │      │
│  │  updates list UI    │  │  updates header     │  │  updates badge      │      │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘      │
│                                                                                 │
│  RESPONSIBILITIES:                                                              │
│  ─────────────────                                                              │
│  • Subscribe to a Query                                                         │
│  • Call its own onStateChange callback when Query changes                       │
│  • Track accessed properties (optimization)                                     │
│  • Manage its own lifecycle (unsubscribe on destroy)                            │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Key Insight:**
```
3 ViewModels calling query(['voices']) =
  • 3 QueryObserver instances (one per ViewModel)
  • 1 Query instance (in QueryCache)
  • 1 HTTP request (deduplicated by Query)
```

**Why v1 was wrong:**
We tried to share QueryObservers between ViewModels. But:
- Deduplication already happens at the Query level
- Each ViewModel needs its own Observer for its own `onStateChange` callback
- Observers are cheap (just subscription + callback)

**Instance Counts:**

| Component | Count | Lifetime |
|-----------|-------|----------|
| QueryClient | 1 | App lifetime (singleton) |
| QueryCache | 1 | Owned by QueryClient |
| Query ['voices'] | 1 | Until garbage collected (gcTime) |
| QueryObserver per ViewModel | 1 each | ViewModel lifetime |
| HTTP requests for ['voices'] | 1 | Deduplicated |

### 4.2 Why onStateChange? The Bridge Between Worlds

#### The Core Problem

React Query uses React's rendering model:
```typescript
// React: PULL model
function VoicesScreen() {
  const { data, isLoading } = useQuery({ queryKey: ['voices'], queryFn })
  //      ^^^^ React "pulls" data on every render
  //           useSyncExternalStore triggers re-render when data changes
  
  return <List data={data} />
}
```

MVVM uses a PUSH model:
```typescript
// MVVM: PUSH model
class VoicesViewModel {
  state$ = new BehaviorSubject<State>({ voices: [], isLoading: true })
  //       ^^^^ We need to PUSH updates into this
  
  // How do we get TanStack Query updates INTO state$?
}
```

#### The Solution: onStateChange

`onStateChange` is our bridge - it's called on **EVERY** query state change:

```typescript
this.query({
  queryKey: ['voices'],
  queryFn: fetchVoices,
  
  onStateChange: (result) => {
    // This fires on EVERY state change:
    // • Initial fetch starts (status: 'pending', isFetching: true)
    // • Fetch completes (status: 'success', data: [...])
    // • Background refetch starts (isFetching: true, data: [old])
    // • Background refetch completes (data: [new])
    // • Error occurs (status: 'error', error: Error)
    // • Retry attempt (failureCount increments)
    // • Window focus refetch
    // • Network reconnect refetch
    // • Manual invalidation
    
    this.updateState({
      voices: result.data ?? [],
      isLoading: result.isLoading,
      isFetching: result.isFetching,
      error: result.error?.message ?? null,
    })
  }
})
```

#### Timeline Example

```
User opens Voices screen:
═══════════════════════════════════════════════════════════════════════════════

t=0ms      │ query() called
           │ → onStateChange({ status: 'pending', isFetching: true, data: undefined })
           │ → UI shows loading spinner
           │
t=200ms    │ HTTP response arrives
           │ → onStateChange({ status: 'success', isFetching: false, data: [...voices] })
           │ → UI shows voice list
           │
t=5min     │ User returns to app (window focus)
           │ Data is stale (staleTime: 5min passed)
           │ → onStateChange({ isFetching: true, data: [...oldVoices] })
           │ → UI shows list + subtle refresh indicator (stale-while-revalidate!)
           │
t=5min+200 │ Background refresh completes
           │ → onStateChange({ isFetching: false, data: [...newVoices] })
           │ → UI seamlessly updates with new data
           │
t=10min    │ Another ViewModel calls invalidateQueries(['voices'])
           │ → onStateChange({ isFetching: true, data: [...currentVoices] })
           │ → Refetch triggered, UI stays responsive

═══════════════════════════════════════════════════════════════════════════════
```

#### What Happens WITHOUT onStateChange?

```typescript
// ❌ WRONG: Only get data once
async fetchVoices() {
  const result = await this.query({
    queryKey: ['voices'],
    queryFn: fetchVoices,
    // No onStateChange!
  })
  
  this.updateState({ voices: result })
  
  // Problems:
  // • No loading state (isLoading never set)
  // • Background refetches? UI never updates
  // • Window focus refetch? UI never updates  
  // • Network reconnect? UI never updates
  // • invalidateQueries()? UI never updates
  // • You just have stale data forever
}
```

### 4.3 The defaultQueryOptions() Function

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

### 4.4 getOptimisticResult() vs getCurrentResult()

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

### 4.5 The mount()/unmount() Lifecycle

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

### 4.6 notifyManager.batchCalls()

```typescript
// Without batching:
// Multiple rapid updates → multiple callbacks → multiple state updates
Query changes → callback() → updateState()
Query changes → callback() → updateState()
Query changes → callback() → updateState()
// Result: 3 state emissions, 3 potential re-renders

// With batching:
observer.subscribe(notifyManager.batchCalls(callback))
// Multiple rapid updates → single batched callback
Query changes ─┐
Query changes ─┼─► batchCalls() → single callback()
Query changes ─┘
// Result: 1 state emission
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
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  INITIALIZATION FLOW                                                            │
│  ════════════════════                                                           │
│                                                                                 │
│  1. App Startup                                                                 │
│     └──► initializeQueryClient() → client.mount()                               │
│                                                                                 │
│  2. Screen Mounts → ViewModel Created                                           │
│     └──► ViewModel.initialize()                                                 │
│         └──► this.query(() => options)                                          │
│             └──► QueryCapability.fetch()                                        │
│                 ├── getOptions() evaluates with current state                   │
│                 ├── client.defaultQueryOptions() merges defaults                │
│                 ├── new QueryObserver() created                                 │
│                 ├── getOptimisticResult() → onStateChange() ◄── immediate!      │
│                 ├── observer.subscribe() with batching                          │
│                 ├── observer.updateResult() catches races                       │
│                 └── Entry stored: { getOptions, observer, unsubscribe }         │
│                                                                                 │
│  3. Query fetches data → observer notified → onStateChange()                    │
│     └──► this.updateState({ data })                                             │
│         └──► state$.next() → UI renders                                         │
│                                                                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  DYNAMIC UPDATE FLOW (when state changes)                                       │
│  ═════════════════════════════════════════                                      │
│                                                                                 │
│  1. User Action                                                                 │
│     └──► this.updateState({ page: 2 })                                          │
│                                                                                 │
│  2. BehaviorSubject.next()                                                      │
│     └──► BaseViewModel's state$ subscription fires                              │
│         └──► queryCapability.onStateChange()                                    │
│                                                                                 │
│  3. For each query entry:                                                       │
│     ├── getOptions() re-evaluates (now returns page: 2)                         │
│     ├── defaultQueryOptions() applies defaults                                  │
│     └── observer.setOptions() → TanStack handles it                             │
│                                                                                 │
│  4. TanStack internally:                                                        │
│     ├── Compares old vs new options                                             │
│     ├── If queryKey changed → switches to different Query                       │
│     ├── If enabled changed → starts/stops fetching                              │
│     └── Triggers fetch if needed → notifies our callback                        │
│                                                                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  CLEANUP FLOW                                                                   │
│  ════════════                                                                   │
│                                                                                 │
│  Screen Unmounts → ViewModel.destroy()                                          │
│  ├── queryCapability.destroy()                                                  │
│  │   └── For each entry: unsubscribe()                                          │
│  └── state$.complete()                                                          │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 5.3 File Structure

```
core/
├── query/
│   ├── index.ts              # Public exports
│   ├── queryClient.ts        # Singleton management (~50 lines)
│   ├── QueryCapability.ts    # Core adapter (~180 lines)
│   └── types.ts              # Type definitions (~80 lines)
│
├── viewmodels/
│   └── BaseViewModel.ts      # Integration (~30 lines added)
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
 * Callback for combined state changes in queryMultiple()
 */
export type CombinedStateChangeCallback<T extends readonly QueryObserverResult<any, any>[]> = 
  (results: T) => void

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
  CombinedStateChangeCallback,
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
   * Fetch multiple queries in parallel with a combined state callback.
   * 
   * WHY NOT just Promise.all([query1, query2])?
   * - Promise.all doesn't give you combined loading state
   * - Each query has independent cache/staleTime
   * - Combined callback fires when ANY query updates
   * 
   * @example
   * await this.queryMultiple([
   *   {
   *     queryKey: ['voices'],
   *     queryFn: fetchVoices,
   *   },
   *   {
   *     queryKey: ['settings'],
   *     queryFn: fetchSettings,
   *   }
   * ], (results) => {
   *   const [voicesResult, settingsResult] = results
   *   this.updateState({
   *     voices: voicesResult.data ?? [],
   *     settings: settingsResult.data ?? null,
   *     isLoading: results.some(r => r.isLoading),
   *   })
   * })
   */
  async fetchMultiple<T extends readonly QueryOptions<any, any, any>[]>(
    optionsArray: T,
    onCombinedStateChange?: CombinedStateChangeCallback<{
      [K in keyof T]: QueryObserverResult<
        T[K] extends QueryOptions<infer D, any, any> ? D : unknown,
        T[K] extends QueryOptions<any, infer E, any> ? E : Error
      >
    }>
  ): Promise<void> {
    this.ensureNotDestroyed()
    
    // Wrap each query's onStateChange to also call the combined callback
    const wrappedOptions = optionsArray.map((options, index) => ({
      ...options,
      onStateChange: (result: QueryObserverResult<any, any>) => {
        // Call individual callback if provided
        options.onStateChange?.(result)
        
        // Call combined callback with all current results
        if (onCombinedStateChange) {
          const allResults = this.queries
            .slice(-optionsArray.length)  // Get the queries we just added
            .map(entry => entry.observer.getCurrentResult())
          
          onCombinedStateChange(allResults as any)
        }
      }
    }))
    
    // Fetch all in parallel
    await Promise.all(wrappedOptions.map(options => this.fetch(options)))
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
  QueryOptions,
  QueryOptionsOrGetter,
  MutationOptions,
  MutationHandle,
  CombinedStateChangeCallback,
  QueryObserverResult,
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
   * Fetch multiple queries in parallel with combined state handling.
   * 
   * @example
   * await this.queryMultiple([
   *   { queryKey: ['voices'], queryFn: fetchVoices },
   *   { queryKey: ['settings'], queryFn: fetchSettings },
   *   { queryKey: ['user'], queryFn: fetchUser },
   * ], (results) => {
   *   const [voices, settings, user] = results
   *   this.updateState({
   *     voices: voices.data ?? [],
   *     settings: settings.data ?? null,
   *     user: user.data ?? null,
   *     isLoading: results.some(r => r.isLoading),
   *   })
   * })
   */
  protected queryMultiple<T extends readonly QueryOptions<any, any, any>[]>(
    optionsArray: T,
    onCombinedStateChange?: CombinedStateChangeCallback<{
      [K in keyof T]: QueryObserverResult<
        T[K] extends QueryOptions<infer D, any, any> ? D : unknown,
        T[K] extends QueryOptions<any, infer E, any> ? E : Error
      >
    }>
  ): Promise<void> {
    return this.__queryCapability.fetchMultiple(optionsArray, onCombinedStateChange)
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
  CombinedStateChangeCallback,
} from './types'

// Re-export queryOptions for type-safe query factories
export { queryOptions } from '@tanstack/query-core'
```

---

## 7. Query Options & Factories

This section covers patterns that improve developer experience, type safety, and code organization.

### 7.1 The queryOptions() Helper

#### The Problem: Silent Typos

```typescript
// ❌ WITHOUT queryOptions - typos NOT caught
const voicesQuery = {
  queryKey: ['voices'],
  queryFn: fetchVoices,
  stalTime: 5000,      // TYPO! Should be 'staleTime'
  retryOnMount: true,  // TYPO! Should be 'refetchOnMount'
}
// TypeScript: "Looks fine to me!" 😱
```

#### The Solution: queryOptions()

```typescript
// ✅ WITH queryOptions - typos CAUGHT
import { queryOptions } from '@tanstack/query-core'

const voicesQuery = queryOptions({
  queryKey: ['voices'],
  queryFn: fetchVoices,
  stalTime: 5000,      // TS Error: Did you mean 'staleTime'?
  retryOnMount: true,  // TS Error: Did you mean 'refetchOnMount'?
})
```

#### Bonus: Type-Safe Cache Access

The real magic of `queryOptions()` is the **DataTag** - it tags the queryKey with the return type:

```typescript
const voicesQuery = queryOptions({
  queryKey: ['voices'],
  queryFn: fetchVoices,  // Returns Promise<Voice[]>
})

// The queryKey is now TAGGED with the return type
voicesQuery.queryKey  // Type: ['voices'] & { [dataTagSymbol]: Voice[] }

// So cache access is type-safe:
const voices = this.getQueryData(voicesQuery.queryKey)
//    ^? Voice[] | undefined  ← INFERRED! No manual type annotation!

// Compare to without queryOptions:
const voices = this.getQueryData<Voice[]>(['voices'])
//                               ^^^^^^^^ Manual type, can be wrong!
```

#### How It Works (Simplified)

```typescript
// queryOptions is essentially an identity function with type magic
function queryOptions<TQueryFnData, TError, TData, TQueryKey extends QueryKey>(
  options: QueryOptions<TQueryFnData, TError, TData, TQueryKey>
): QueryOptions<TQueryFnData, TError, TData, TQueryKey> & {
  queryKey: TQueryKey & { [dataTagSymbol]: TQueryFnData }  // ◄── The magic
} {
  return options as any  // Runtime: just returns the same object
}

// At runtime: identity function (no overhead)
// At type level: adds DataTag to queryKey for type inference
```

### 7.2 Query Factory Pattern

For larger apps, organize queries into **factory objects** with hierarchical keys.

#### Basic Structure

```typescript
// queries/voiceQueries.ts

import { queryOptions } from '@tanstack/query-core'
import { VoiceAPI } from '../api/VoiceAPI'
import type { Voice, VoiceFilters } from '../types'

export const voiceQueries = {
  // ─────────────────────────────────────────────────────────────
  // Key-only entries (for invalidation, no fetch)
  // ─────────────────────────────────────────────────────────────
  
  all: () => ['voices'] as const,
  //         ^^^^^^^^^
  //         Base key - invalidating this invalidates EVERYTHING below
  
  lists: () => [...voiceQueries.all(), 'list'] as const,
  //           ['voices', 'list']
  //           Invalidating this invalidates all lists (with any filters)
  
  details: () => [...voiceQueries.all(), 'detail'] as const,
  //             ['voices', 'detail']
  //             Invalidating this invalidates all detail queries
  
  // ─────────────────────────────────────────────────────────────
  // Full query options (for fetching)
  // ─────────────────────────────────────────────────────────────
  
  list: (filters?: VoiceFilters) =>
    queryOptions({
      queryKey: [...voiceQueries.lists(), filters] as const,
      //        ['voices', 'list', { search: 'hello', page: 2 }]
      queryFn: () => VoiceAPI.getVoices(filters),
      staleTime: 5 * 60 * 1000,  // 5 minutes
    }),
  
  detail: (id: string) =>
    queryOptions({
      queryKey: [...voiceQueries.details(), id] as const,
      //        ['voices', 'detail', 'voice-123']
      queryFn: () => VoiceAPI.getVoice(id),
      staleTime: 10 * 60 * 1000,  // 10 minutes
    }),
}
```

#### Key Hierarchy Visualization

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  voiceQueries.all()     →  ['voices']                                           │
│       │                                                                         │
│       ├── voiceQueries.lists()   →  ['voices', 'list']                          │
│       │        │                                                                │
│       │        ├── voiceQueries.list()                                          │
│       │        │   →  ['voices', 'list', undefined]                             │
│       │        │                                                                │
│       │        ├── voiceQueries.list({ search: 'hello' })                       │
│       │        │   →  ['voices', 'list', { search: 'hello' }]                   │
│       │        │                                                                │
│       │        └── voiceQueries.list({ page: 2, search: 'hello' })              │
│       │            →  ['voices', 'list', { page: 2, search: 'hello' }]          │
│       │                                                                         │
│       └── voiceQueries.details() →  ['voices', 'detail']                        │
│                │                                                                │
│                ├── voiceQueries.detail('voice-1')                               │
│                │   →  ['voices', 'detail', 'voice-1']                           │
│                │                                                                │
│                └── voiceQueries.detail('voice-2')                               │
│                    →  ['voices', 'detail', 'voice-2']                           │
│                                                                                 │
│  INVALIDATION EXAMPLES:                                                         │
│  ──────────────────────                                                         │
│                                                                                 │
│  invalidateQueries({ queryKey: voiceQueries.all() })                            │
│  → Invalidates EVERYTHING: all lists, all details                               │
│                                                                                 │
│  invalidateQueries({ queryKey: voiceQueries.lists() })                          │
│  → Invalidates all lists (any filters), but NOT details                         │
│                                                                                 │
│  invalidateQueries({ queryKey: voiceQueries.details() })                        │
│  → Invalidates all details, but NOT lists                                       │
│                                                                                 │
│  invalidateQueries({ queryKey: voiceQueries.detail('voice-1').queryKey })       │
│  → Invalidates only voice-1 detail                                              │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 7.3 Using Query Factories in ViewModels

```typescript
import { voiceQueries } from '../queries/voiceQueries'

class VoicesViewModel extends BaseViewModel<VoicesState> {
  
  // ─────────────────────────────────────────────────────────────
  // Fetching with query factory
  // ─────────────────────────────────────────────────────────────
  
  async fetchVoices(filters?: VoiceFilters) {
    await this.query({
      ...voiceQueries.list(filters),  // ◄── Spread the options!
      
      onStateChange: (result) => {
        this.updateState({
          voices: result.data ?? [],
          isFetching: result.isFetching,
          error: result.error?.message ?? null,
        })
      }
    })
  }
  
  async fetchVoiceDetail(id: string) {
    await this.query({
      ...voiceQueries.detail(id),  // ◄── Spread the options!
      
      onStateChange: (result) => {
        this.updateState({
          selectedVoice: result.data ?? null,
        })
      }
    })
  }
  
  // ─────────────────────────────────────────────────────────────
  // Type-safe cache access
  // ─────────────────────────────────────────────────────────────
  
  getCachedVoice(id: string): Voice | undefined {
    // Type is INFERRED from queryFn return type!
    return this.getQueryData(voiceQueries.detail(id).queryKey)
  }
  
  getCachedVoices(): Voice[] | undefined {
    return this.getQueryData(voiceQueries.list().queryKey)
  }
  
  // ─────────────────────────────────────────────────────────────
  // Hierarchical invalidation
  // ─────────────────────────────────────────────────────────────
  
  async invalidateAllVoices() {
    // Invalidates: all lists AND all details
    await this.invalidateQueries({ queryKey: voiceQueries.all() })
  }
  
  async invalidateVoiceLists() {
    // Invalidates: all lists (with any filters), NOT details
    await this.invalidateQueries({ queryKey: voiceQueries.lists() })
  }
  
  async invalidateVoiceDetail(id: string) {
    // Invalidates: only this specific detail
    await this.invalidateQueries({ 
      queryKey: voiceQueries.detail(id).queryKey,
      exact: true 
    })
  }
}
```

### 7.4 Multiple Query Factories

For larger apps, create one factory per domain:

```typescript
// queries/index.ts
export { voiceQueries } from './voiceQueries'
export { userQueries } from './userQueries'
export { settingsQueries } from './settingsQueries'

// queries/userQueries.ts
export const userQueries = {
  all: () => ['users'] as const,
  
  current: () =>
    queryOptions({
      queryKey: [...userQueries.all(), 'current'] as const,
      queryFn: () => UserAPI.getCurrent(),
      staleTime: 60 * 1000,
    }),
  
  detail: (id: string) =>
    queryOptions({
      queryKey: [...userQueries.all(), 'detail', id] as const,
      queryFn: () => UserAPI.getById(id),
      staleTime: 5 * 60 * 1000,
    }),
}

// queries/settingsQueries.ts
export const settingsQueries = {
  all: () => ['settings'] as const,
  
  app: () =>
    queryOptions({
      queryKey: [...settingsQueries.all(), 'app'] as const,
      queryFn: () => SettingsAPI.getAppSettings(),
      staleTime: 10 * 60 * 1000,
    }),
  
  user: () =>
    queryOptions({
      queryKey: [...settingsQueries.all(), 'user'] as const,
      queryFn: () => SettingsAPI.getUserSettings(),
      staleTime: 5 * 60 * 1000,
    }),
}
```

### 7.5 Query Factory with Mutations

You can also organize mutations in the factory:

```typescript
// queries/voiceQueries.ts

export const voiceQueries = {
  // ... query options ...
  
  // Mutation helpers (not using queryOptions, just organized here)
  mutations: {
    create: () => ({
      mutationFn: (data: CreateVoiceInput) => VoiceAPI.create(data),
    }),
    
    update: () => ({
      mutationFn: (input: { id: string; data: UpdateVoiceInput }) =>
        VoiceAPI.update(input.id, input.data),
    }),
    
    delete: () => ({
      mutationFn: (id: string) => VoiceAPI.delete(id),
    }),
  },
}

// Usage in ViewModel:
class VoicesViewModel extends BaseViewModel<State> {
  private createMutation = this.mutation({
    ...voiceQueries.mutations.create(),
    
    onSuccess: () => {
      // Invalidate lists after creating
      this.invalidateQueries({ queryKey: voiceQueries.lists() })
    },
    
    onStateChange: (result) => {
      this.updateState({ isCreating: result.isPending })
    },
  })
}
```

### 7.6 Summary: Benefits of Query Factories

| Benefit | Description |
|---------|-------------|
| **Type Safety** | `queryOptions()` catches typos, `getQueryData` infers types |
| **Centralized** | All query definitions in one place per domain |
| **Hierarchical Keys** | Granular invalidation (all, lists only, single detail) |
| **Reusable** | Same factory used across multiple ViewModels |
| **Discoverable** | IDE autocomplete shows all available queries |
| **Consistent** | Enforces naming conventions and staleTime defaults |

---

## 8. Use Cases & Patterns

### 8.1 Basic Query

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

### 8.2 Dynamic Query (State-Dependent)

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

### 8.3 Multiple Parallel Queries

```typescript
interface DashboardState {
  voices: Voice[]
  settings: Settings | null
  user: User | null
  isLoading: boolean
}

class DashboardViewModel extends BaseViewModel<DashboardState> {
  async initialize() {
    await this.queryMultiple([
      {
        queryKey: ['voices'],
        queryFn: VoiceAPI.getAll,
      },
      {
        queryKey: ['settings'],
        queryFn: SettingsAPI.get,
      },
      {
        queryKey: ['user', 'current'],
        queryFn: UserAPI.getCurrent,
      }
    ], (results) => {
      const [voicesResult, settingsResult, userResult] = results
      
      this.updateState({
        voices: voicesResult.data ?? [],
        settings: settingsResult.data ?? null,
        user: userResult.data ?? null,
        isLoading: results.some(r => r.isLoading),
      })
    })
  }
}
```

### 8.4 Dependent Queries

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

### 8.5 Mutation with Optimistic Update

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

### 8.6 Prefetching

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

### 8.7 Cross-ViewModel Cache Invalidation

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

---

## 9. Data Flow Diagrams

### 9.1 First Fetch (Cache Miss)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  User opens Voices screen                                                       │
│                                                                                 │
│  t=0ms    VoicesViewModel.initialize()                                          │
│           │                                                                     │
│           └──► this.query({ queryKey: ['voices'], queryFn, onStateChange })     │
│                     │                                                           │
│                     ▼                                                           │
│           QueryCapability.fetch()                                               │
│                     │                                                           │
│                     ├──► new QueryObserver(client, options)                     │
│                     │                                                           │
│                     ├──► QueryCache.build(['voices'])                           │
│                     │    └──► Query doesn't exist → CREATE new Query            │
│                     │                                                           │
│                     ├──► getOptimisticResult()                                  │
│                     │    └──► { status: 'pending', data: undefined }            │
│                     │                                                           │
│                     └──► onStateChange({ status: 'pending', isFetching: true }) │
│                          └──► UI shows loading spinner                          │
│                                                                                 │
│  t=1ms    Query.fetch() starts                                                  │
│           └──► HTTP GET /api/voices                                             │
│                                                                                 │
│  t=200ms  HTTP Response arrives                                                 │
│           │                                                                     │
│           └──► Query.setState({ data: [...voices], status: 'success' })         │
│                     │                                                           │
│                     └──► Query notifies all observers                           │
│                          │                                                      │
│                          └──► Our observer's callback fires                     │
│                               │                                                 │
│                               └──► onStateChange({                              │
│                                      status: 'success',                         │
│                                      data: [...voices],                         │
│                                      isFetching: false                          │
│                                    })                                           │
│                                    │                                            │
│                                    └──► UI shows voice list                     │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 9.2 Cache Hit (Fresh Data)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  User navigates BACK to Voices screen (within staleTime)                        │
│                                                                                 │
│  t=0ms    VoicesViewModel.initialize()                                          │
│           │                                                                     │
│           └──► this.query({ queryKey: ['voices'], ... })                        │
│                     │                                                           │
│                     ▼                                                           │
│           QueryCapability.fetch()                                               │
│                     │                                                           │
│                     ├──► new QueryObserver(client, options)                     │
│                     │                                                           │
│                     ├──► QueryCache.build(['voices'])                           │
│                     │    └──► Query EXISTS, isStale: false                      │
│                     │                                                           │
│                     ├──► getOptimisticResult()                                  │
│                     │    └──► { status: 'success', data: [...cached] }          │
│                     │                                                           │
│                     └──► onStateChange({ status: 'success', data: [...] })      │
│                          └──► UI shows voice list IMMEDIATELY                   │
│                                                                                 │
│           ════════════════════════════════════════════════════════════          │
│           NO HTTP REQUEST! Data served from cache.                              │
│           Total time: ~0ms                                                      │
│           ════════════════════════════════════════════════════════════          │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 9.3 Stale-While-Revalidate

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  User returns to app after 5 minutes (data is STALE)                            │
│                                                                                 │
│  t=0ms    Window focus event fires                                              │
│           │                                                                     │
│           └──► focusManager notifies QueryClient                                │
│                     │                                                           │
│                     └──► QueryCache.onFocus()                                   │
│                          │                                                      │
│                          └──► For each Query with observers:                    │
│                               Query['voices'].isStale? YES                      │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  STALE-WHILE-REVALIDATE IN ACTION                                       │    │
│  │                                                                         │    │
│  │  Step 1: Show stale data immediately (UI never blank!)                  │    │
│  │  ──────────────────────────────────────────────────                     │    │
│  │  onStateChange({                                                        │    │
│  │    status: 'success',    ◄── Still success!                             │    │
│  │    data: [...oldVoices], ◄── Show cached data                           │    │
│  │    isFetching: true,     ◄── But we ARE fetching                        │    │
│  │    isStale: true                                                        │    │
│  │  })                                                                     │    │
│  │  └──► UI shows list + subtle refresh indicator                          │    │
│  │                                                                         │    │
│  │  Step 2: Background fetch (user doesn't wait)                           │    │
│  │  ────────────────────────────────────────────                           │    │
│  │  HTTP GET /api/voices (in background)                                   │    │
│  │                                                                         │    │
│  │  Step 3: Seamless update when fresh data arrives                        │    │
│  │  ───────────────────────────────────────────────                        │    │
│  │  onStateChange({                                                        │    │
│  │    status: 'success',                                                   │    │
│  │    data: [...newVoices], ◄── Fresh data!                                │    │
│  │    isFetching: false,    ◄── Done fetching                              │    │
│  │    isStale: false                                                       │    │
│  │  })                                                                     │    │
│  │  └──► UI seamlessly updates with new data                               │    │
│  │                                                                         │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                 │
│  USER EXPERIENCE:                                                               │
│  • User sees data immediately (0ms)                                             │
│  • Subtle indicator shows "refreshing..."                                       │
│  • Data updates in-place when ready                                             │
│  • No loading spinner, no blank screen                                          │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 9.4 Cache Sharing Between ViewModels

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  QueryCache (Singleton)                                                         │
│  ═══════════════════════                                                        │
│           │                                                                     │
│           └──► Query ['voices']                                                 │
│                │                                                                │
│                ├── data: Voice[]                                                │
│                ├── status: 'success'                                            │
│                │                                                                │
│                └── observers: [                                                 │
│                      QueryObserver (VoicesListVM),                              │
│                      QueryObserver (VoiceDetailVM),                             │
│                      QueryObserver (SettingsVM)                                 │
│                    ]                                                            │
│                                                                                 │
│  ═══════════════════════════════════════════════════════════════════════════    │
│                                                                                 │
│  SCENARIO: VoicesListVM loads first                                             │
│  ─────────────────────────────────────                                          │
│                                                                                 │
│  t=0ms    VoicesListVM.query(['voices'])                                        │
│           └──► Creates Query, HTTP request starts                               │
│                                                                                 │
│  t=100ms  VoiceDetailVM.query(['voices']) ◄── SAME KEY!                         │
│           └──► Attaches to EXISTING Query                                       │
│           └──► Gets current state immediately (pending)                         │
│           └──► NO new HTTP request!                                             │
│                                                                                 │
│  t=200ms  HTTP response arrives                                                 │
│           └──► Query notifies ALL observers:                                    │
│               ├──► VoicesListVM.onStateChange({ data: [...] })                  │
│               └──► VoiceDetailVM.onStateChange({ data: [...] })                 │
│                                                                                 │
│  RESULT:                                                                        │
│  • 1 HTTP request (not 2)                                                       │
│  • Both ViewModels get data simultaneously                                      │
│  • Future refetch updates BOTH automatically                                    │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Edge Cases & Error Handling

### 10.1 Destroy During Fetch

**Scenario:** User navigates away while a fetch is in progress.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  t=0ms    VoicesViewModel.initialize()                                          │
│           └──► this.query(['voices']) → HTTP request starts                     │
│                                                                                 │
│  t=100ms  User presses BACK button                                              │
│           └──► VoicesViewModel.destroy() called                                 │
│               │                                                                 │
│               ├──► queryCapability.isDestroyed = true                           │
│               └──► unsubscribe() called for all observers                       │
│                                                                                 │
│  t=200ms  HTTP response arrives                                                 │
│           └──► Query updates its state                                          │
│           └──► Query tries to notify observers                                  │
│               │                                                                 │
│               └──► Our subscription callback:                                   │
│                    if (this.isDestroyed) return  ◄── NO-OP!                     │
│                                                                                 │
│  RESULT:                                                                        │
│  • No crash                                                                     │
│  • No memory leak                                                               │
│  • No "setState on unmounted component" error                                   │
│  • Data is still cached for other ViewModels!                                   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**The code that handles this:**
```typescript
// In QueryCapability.fetch()
const unsubscribe = observer.subscribe(
  notifyManager.batchCalls(() => {
    if (this.isDestroyed) return  // ◄── Safe guard
    // ...
  })
)

// In QueryCapability.destroy()
destroy(): void {
  this.isDestroyed = true  // ◄── Set flag FIRST
  for (const entry of this.queries) {
    entry.unsubscribe()  // ◄── Then unsubscribe
  }
}
```

### 10.2 Same Query Called Multiple Times

**Scenario:** `fetchVoices()` called 3 times rapidly (button mashing, navigation race).

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  t=0ms    this.query(['voices'])  → QueryEntry 1 created, Observer 1            │
│  t=1ms    this.query(['voices'])  → QueryEntry 2 created, Observer 2            │
│  t=2ms    this.query(['voices'])  → QueryEntry 3 created, Observer 3            │
│                                                                                 │
│  QueryCache:                                                                    │
│  └──► Query ['voices'] (ONE Query)                                              │
│       └──► observers: [Observer 1, Observer 2, Observer 3]                      │
│       └──► HTTP requests: 1 (DEDUPLICATED!)                                     │
│                                                                                 │
│  t=200ms  HTTP response arrives                                                 │
│           └──► All 3 observers notified                                         │
│           └──► All 3 onStateChange callbacks fire                               │
│                                                                                 │
│  MEMORY: 3 QueryEntry objects (minor overhead)                                  │
│  HTTP: 1 request (deduplicated by Query)                                        │
│  CORRECTNESS: All callbacks receive same data ✓                                 │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Prevention pattern:**
```typescript
class VoicesViewModel extends BaseViewModel<State> {
  private voicesQueryInitialized = false
  
  async initialize() {
    if (this.voicesQueryInitialized) return  // Guard against double-init
    this.voicesQueryInitialized = true
    
    await this.query({ queryKey: ['voices'], ... })
  }
}
```

### 10.3 Query Key Changes (Dynamic Queries)

**Scenario:** User changes page, queryKey changes.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  // Dynamic query setup                                                         │
│  this.query(() => ({                                                            │
│    queryKey: ['voices', this.getStateValue().page],                             │
│    queryFn: () => fetchVoices(this.getStateValue().page),                       │
│  }))                                                                            │
│                                                                                 │
│  ═══════════════════════════════════════════════════════════════════════════    │
│                                                                                 │
│  t=0      Page 1 loaded                                                         │
│           queryKey: ['voices', 1]                                               │
│           Query ['voices', 1] created and fetched                               │
│                                                                                 │
│  t=10s    User clicks "Page 2"                                                  │
│           └──► this.updateState({ page: 2 })                                    │
│               │                                                                 │
│               └──► state$.next() fires                                          │
│                   │                                                             │
│                   └──► queryCapability.onStateChange()                          │
│                       │                                                         │
│                       └──► For each entry:                                      │
│                            options = entry.getOptions()                         │
│                            // Now returns: queryKey: ['voices', 2]              │
│                            │                                                    │
│                            └──► observer.setOptions(newOptions)                 │
│                                │                                                │
│                                └──► TanStack INTERNALLY:                        │
│                                    • Detects queryKey changed                   │
│                                    • Unsubscribes from Query ['voices', 1]      │
│                                    • Subscribes to Query ['voices', 2]          │
│                                    • Creates Query ['voices', 2] if new         │
│                                    • Triggers fetch for page 2                  │
│                                                                                 │
│  BONUS: Query ['voices', 1] is still cached!                                    │
│         If user clicks "Page 1" again → instant cache hit                       │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 10.4 Network Errors and Retries

**Scenario:** Server returns 500 error.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  Default retry behavior (retry: 3)                                              │
│                                                                                 │
│  t=0ms     Attempt 1: HTTP GET /api/voices                                      │
│  t=200ms   Response: 500 Internal Server Error                                  │
│            └──► onStateChange({                                                 │
│                   status: 'pending',     ◄── Still pending, not error yet       │
│                   isFetching: true,                                             │
│                   failureCount: 1,       ◄── Can show "Retrying..."             │
│                   failureReason: Error                                          │
│                 })                                                              │
│                                                                                 │
│  t=1200ms  Attempt 2: HTTP GET /api/voices (after 1s delay)                     │
│  t=1400ms  Response: 500 Internal Server Error                                  │
│            └──► onStateChange({ failureCount: 2 })                              │
│                                                                                 │
│  t=3400ms  Attempt 3: HTTP GET /api/voices (after 2s delay)                     │
│  t=3600ms  Response: 500 Internal Server Error                                  │
│            └──► onStateChange({ failureCount: 3 })                              │
│                                                                                 │
│  t=7600ms  Attempt 4: HTTP GET /api/voices (after 4s delay)                     │
│  t=7800ms  Response: 500 Internal Server Error                                  │
│            └──► GIVE UP (retry limit reached)                                   │
│            └──► onStateChange({                                                 │
│                   status: 'error',       ◄── NOW it's error                     │
│                   error: Error,                                                 │
│                   isFetching: false,                                            │
│                   failureCount: 4                                               │
│                 })                                                              │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Custom retry logic:**
```typescript
this.query({
  queryKey: ['voices'],
  queryFn: fetchVoices,
  
  retry: (failureCount, error) => {
    // Don't retry client errors (4xx)
    if (error.status >= 400 && error.status < 500) {
      return false
    }
    // Retry server errors up to 3 times
    return failureCount < 3
  },
  
  retryDelay: (attemptIndex) => {
    // Exponential backoff: 1s, 2s, 4s, 8s...
    return Math.min(1000 * 2 ** attemptIndex, 30000)
  },
  
  onStateChange: (result) => {
    if (result.failureCount > 0 && result.isFetching) {
      this.updateState({ 
        retryMessage: `Retrying... (attempt ${result.failureCount + 1}/4)` 
      })
    }
    // ...
  }
})
```

### 10.5 Enabled/Disabled Queries (Dependent)

**Scenario:** Query B depends on Query A's result.

```typescript
// Query B: Posts for a user (depends on user existing)
this.query(() => {
  const user = this.getStateValue().user
  
  return {
    queryKey: ['posts', user?.id],
    queryFn: () => fetchPosts(user!.id),
    enabled: !!user,  // Only run when user exists
    onStateChange: (result) => {
      this.updateState({ posts: result.data ?? [] })
    }
  }
})
```

**Timeline:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  t=0ms    state.user = null                                                     │
│           Query B options: { enabled: false }                                   │
│           └──► QueryObserver created but:                                       │
│               • fetchStatus: 'idle' (NOT 'fetching')                            │
│               • NO HTTP request                                                 │
│               • onStateChange({ status: 'pending', fetchStatus: 'idle' })       │
│                                                                                 │
│  t=200ms  User query completes                                                  │
│           └──► this.updateState({ user: { id: '123', name: 'Alice' } })         │
│               │                                                                 │
│               └──► state$ fires                                                 │
│                   │                                                             │
│                   └──► queryCapability.onStateChange()                          │
│                       │                                                         │
│                       └──► Query B: getOptions() returns { enabled: true }      │
│                           │                                                     │
│                           └──► observer.setOptions({ enabled: true })           │
│                               │                                                 │
│                               └──► TanStack sees enabled changed                │
│                                   └──► TRIGGERS FETCH automatically!            │
│                                                                                 │
│  t=400ms  Posts query completes                                                 │
│           └──► onStateChange({ status: 'success', data: [...posts] })           │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 10.6 Race Conditions: Invalidation During Fetch

**Scenario:** Query is invalidated while a fetch is already in progress.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  t=0ms    User opens screen                                                     │
│           └──► query(['voices']) → Fetch A starts                               │
│                                                                                 │
│  t=50ms   Another ViewModel calls invalidateQueries(['voices'])                 │
│           └──► Query is marked invalid                                          │
│           └──► But Fetch A is still in progress!                                │
│           └──► TanStack: "I'll refetch after current fetch completes"           │
│                                                                                 │
│  t=200ms  Fetch A completes                                                     │
│           └──► Data is stored                                                   │
│           └──► onStateChange({ data: [...] }) fires                             │
│           └──► Query is still marked invalid!                                   │
│           └──► TanStack: "Query invalid, triggering Fetch B"                    │
│                                                                                 │
│  t=400ms  Fetch B completes (fresh data after invalidation)                     │
│           └──► onStateChange({ data: [...fresh] }) fires                        │
│                                                                                 │
│  RESULT:                                                                        │
│  • User sees data from Fetch A (t=200ms) - good UX                              │
│  • Fresh data from Fetch B replaces it (t=400ms) - correct data                 │
│  • No race condition, TanStack handles this internally                          │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 11. Testing Strategy

### 11.1 Unit Testing QueryCapability

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

### 11.2 Integration Testing ViewModel

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

## 12. Platform Configuration

### 12.1 React Native Setup

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

### 12.2 Web Setup

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

## 13. Migration Guide

### 13.1 From Manual Fetching

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

### 13.2 Checklist

- [ ] Install `@tanstack/query-core`
- [ ] Add QueryCapability, types, queryClient to your project
- [ ] Update BaseViewModel with query methods
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
| onStateChange callback | Bridges TanStack's PULL model to MVVM's PUSH model |

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
| TypeScript | Full inference via TanStack types + queryOptions() |
| Hierarchical Invalidation | Via query factories with key prefixes |

### Code Stats

| File | Lines | Purpose |
|------|-------|---------|
| types.ts | ~100 | Type definitions |
| queryClient.ts | ~60 | Singleton management |
| QueryCapability.ts | ~200 | Core adapter |
| BaseViewModel.ts | +35 | Integration |
| **Total new code** | **~395** | Full TanStack Query integration |

---

*v5 Final - Complete architecture with comprehensive documentation*
