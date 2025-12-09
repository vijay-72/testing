# TanStack Query MVVM Adapter: Architecture & Core Concepts

## Document Purpose

This document explains the **architecture, design decisions, and core concepts** behind integrating TanStack Query into MVVM-based applications. It covers the "why" and "how" at a conceptual level.

For implementation details and code, see the companion document: `tanstack-query-mvvm-implementation.md`

---

## Table of Contents

1. [Problem Statement & Requirements](#1-problem-statement--requirements)
2. [Why TanStack Query?](#2-why-tanstack-query)
3. [Architecture Evolution: Lessons Learned](#3-architecture-evolution-lessons-learned)
4. [Core Concepts You Must Understand](#4-core-concepts-you-must-understand)
5. [Data Flow Architecture](#5-data-flow-architecture)
6. [How Parallel Queries Work](#6-how-parallel-queries-work)
7. [Type System Design](#7-type-system-design)

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
| **Parallel Queries** | Fetch multiple resources simultaneously | High |

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

### 2.1 Framework-Agnostic Core

TanStack Query (formerly React Query) is a **framework-agnostic** async state management library. The core logic lives in `@tanstack/query-core`, while framework adapters provide idiomatic APIs.

**Key Insight:** React Query, Vue Query, Solid Query are all thin adapters on top of the same core. We can build our own MVVM adapter.

### 2.2 The Hub-and-Spoke Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │       @tanstack/query-core                  │
                    │                                             │
                    │  QueryClient, QueryObserver,                │
                    │  QueriesObserver, QueryCache,               │
                    │  MutationObserver, focusManager,            │
                    │  onlineManager, notifyManager               │
                    └────────────────────┬────────────────────────┘
                                         │
         ┌───────────────────┬───────────┼───────────┬───────────────────┐
         │                   │           │           │                   │
         ▼                   ▼           ▼           ▼                   ▼
    ┌─────────┐       ┌─────────┐ ┌─────────┐ ┌─────────┐       ┌─────────┐
    │  React  │       │   Vue   │ │  Solid  │ │ Svelte  │       │  MVVM   │
    │ Adapter │       │ Adapter │ │ Adapter │ │ Adapter │       │ Adapter │
    │         │       │         │ │         │ │         │       │ (Ours)  │
    │useQuery │       │useQuery │ │ create  │ │ create  │       │ this.   │
    │   ()    │       │   ()    │ │ Query() │ │ Query() │       │ query() │
    └─────────┘       └─────────┘ └─────────┘ └─────────┘       └─────────┘
```

### 2.3 Core Capabilities

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
│  PARALLEL QUERIES                │  PERSISTENCE (Optional)                      │
│  ────────────────                │  ───────────────────────                     │
│  • QueriesObserver for batching  │  • localStorage                              │
│  • Combined state management     │  • IndexedDB                                 │
│  • Dynamic query arrays          │  • AsyncStorage (React Native)               │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Architecture Evolution: Lessons Learned

Understanding our mistakes is crucial to understanding why the final design is the way it is.

### 3.1 Version 1: Shared Observers (FAILED)

**What we tried:**
```typescript
class QueryCapability {
  // WRONG: Tried to reuse observers by query key
  private observers = new Map<string, QueryObserver>()
  
  async fetch(options) {
    const hash = hashKey(options.queryKey)
    let observer = this.observers.get(hash)
    
    if (!observer) {
      observer = new QueryObserver(...)
      this.observers.set(hash, observer)
    }
    // Tried to swap callbacks when same query called again
  }
}
```

**Why it failed:**
1. **Closure Capture Bug**: The subscription callback captured the wrong `onStateChange`
2. **"First Callback Wins"**: Only the first caller's callback ever got called
3. **State Desynchronization**: Different ViewModels got out of sync

**The core misunderstanding:** We thought sharing observers would be "efficient." But deduplication happens at the **Query** level, not the Observer level. Observers are cheap - they just subscribe and notify.

### 3.2 Version 2: Missing Defaults

**What we tried:**
```typescript
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

**The core misunderstanding:** We thought we knew what options mattered. TanStack Query has a sophisticated defaults system that merges global defaults, query-specific defaults, and provided options.

### 3.3 Version 3: Over-Engineered (QueryHandle Class)

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

**The core misunderstanding:** We thought we needed to optimize by checking if options changed. But official adapters just call `observer.setOptions()` every time and let TanStack handle it internally.

### 3.4 Final Version: Match Official Adapters (CORRECT)

**The realization:** Stop being clever. Copy what works.

From React's `useBaseQuery`:
```typescript
// React Query - NO hashing, just call setOptions
React.useEffect(() => {
  observer.setOptions(defaultedOptions)
}, [defaultedOptions, observer])
```

**Final Design Principles:**
1. One observer per `query()` call (not shared)
2. Always use `defaultQueryOptions()` (get all defaults)
3. Just call `setOptions()` - TanStack handles comparison
4. No QueryHandle class, no custom hashing
5. Match the official adapters exactly

---

## 4. Core Concepts You Must Understand

### 4.1 The Query vs Observer Distinction

This is the **most important concept** to understand:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  QueryCache (Singleton - lives for app lifetime)                                │
│  ═══════════════════════════════════════════════                                │
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

### 4.2 Why onStateChange? The Bridge Between Worlds

#### The Core Problem

React Query uses React's rendering model (PULL):
```typescript
function VoicesScreen() {
  const { data, isLoading } = useQuery({ queryKey: ['voices'], queryFn })
  //      ^^^^ React "pulls" data on every render
  return <List data={data} />
}
```

MVVM uses a PUSH model:
```typescript
class VoicesViewModel {
  state$ = new BehaviorSubject<State>({ voices: [], isLoading: true })
  //       ^^^^ We need to PUSH updates into this
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

---

## 5. Data Flow Architecture

### 5.1 Single Query Flow (this.query)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  INITIALIZATION FLOW                                                            │
│  ═══════════════════                                                            │
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
│                 └── Entry stored: { getOptions, observer, unsubscribe }         │
│                                                                                 │
│  3. Query fetches data → observer notified → onStateChange()                    │
│     └──► this.updateState({ data })                                             │
│         └──► state$.next() → UI renders                                         │
│                                                                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  DYNAMIC UPDATE FLOW (when state changes)                                       │
│  ════════════════════════════════════════                                       │
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

### 5.2 Stale-While-Revalidate Pattern

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  User returns to app after 5 minutes (data is STALE)                            │
│                                                                                 │
│  t=0ms    Window focus event fires                                              │
│           └──► focusManager notifies QueryClient                                │
│               └──► QueryCache.onFocus()                                         │
│                   └──► Query['voices'].isStale? YES → refetch                   │
│                                                                                 │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │  STALE-WHILE-REVALIDATE IN ACTION                                         │  │
│  │                                                                           │  │
│  │  Step 1: Show stale data immediately (UI never blank!)                    │  │
│  │  ────────────────────────────────────────────────────                     │  │
│  │  onStateChange({                                                          │  │
│  │    status: 'success',    ◄── Still success!                               │  │
│  │    data: [...oldVoices], ◄── Show cached data                             │  │
│  │    isFetching: true,     ◄── But we ARE fetching                          │  │
│  │    isStale: true                                                          │  │
│  │  })                                                                       │  │
│  │  └──► UI shows list + subtle refresh indicator                            │  │
│  │                                                                           │  │
│  │  Step 2: Background fetch (user doesn't wait)                             │  │
│  │  ────────────────────────────────────────────                             │  │
│  │  HTTP GET /api/voices (in background)                                     │  │
│  │                                                                           │  │
│  │  Step 3: Seamless update when fresh data arrives                          │  │
│  │  ───────────────────────────────────────────────                          │  │
│  │  onStateChange({                                                          │  │
│  │    status: 'success',                                                     │  │
│  │    data: [...newVoices], ◄── Fresh data!                                  │  │
│  │    isFetching: false,    ◄── Done fetching                                │  │
│  │    isStale: false                                                         │  │
│  │  })                                                                       │  │
│  │  └──► UI seamlessly updates with new data                                 │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  USER EXPERIENCE:                                                               │
│  • User sees data immediately (0ms)                                             │
│  • Subtle indicator shows "refreshing..."                                       │
│  • Data updates in-place when ready                                             │
│  • No loading spinner, no blank screen                                          │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 5.3 Cache Sharing Between ViewModels

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  QueryCache (Singleton)                                                         │
│  ══════════════════════                                                         │
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

## 6. How Parallel Queries Work

### 6.1 The Problem with Sequential Fetching

```typescript
// BAD: Sequential fetching
async initialize() {
  const users = await fetchUsers()     // 300ms
  const posts = await fetchPosts()     // 100ms  
  const settings = await fetchSettings() // 200ms
}
// Total: 600ms (sum of all)
```

### 6.2 The Solution: Parallel Queries

TanStack Query provides two ways to fetch in parallel:

**Option 1: Multiple useQuery/this.query calls**
```typescript
// Each query fetches independently
await this.query({ queryKey: ['users'], ... })
await this.query({ queryKey: ['posts'], ... })
await this.query({ queryKey: ['settings'], ... })
```
- ✅ Queries run in parallel
- ❌ Hard to get combined loading state
- ❌ Each has separate subscription

**Option 2: useQueries/this.queryMultiple (QueriesObserver)**
```typescript
await this.queryMultiple({
  queries: [
    { queryKey: ['users'], queryFn: fetchUsers },
    { queryKey: ['posts'], queryFn: fetchPosts },
    { queryKey: ['settings'], queryFn: fetchSettings },
  ],
  onStateChange: (results) => {
    // Single callback with ALL results
    this.updateState({
      isLoading: results.some(r => r.status === 'pending'),
    })
  }
})
```
- ✅ Queries run in parallel
- ✅ Single combined callback
- ✅ Easy to derive combined states

### 6.3 QueriesObserver vs Multiple QueryObservers

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  WRONG: Multiple QueryObservers (what we had before)                            │
│  ═══════════════════════════════════════════════════                            │
│                                                                                 │
│  QueryObserver 1 ──► subscription callback 1 ──► onStateChange1()               │
│  QueryObserver 2 ──► subscription callback 2 ──► onStateChange2()               │
│  QueryObserver 3 ──► subscription callback 3 ──► onStateChange3()               │
│                                                                                 │
│  Problem: Getting "all results" requires manual coordination                    │
│           Race conditions possible with manual assembly                         │
│                                                                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  CORRECT: QueriesObserver (single observer for all)                             │
│  ══════════════════════════════════════════════════                             │
│                                                                                 │
│  QueriesObserver ──► single subscription ──► onStateChange(ALL results)         │
│       │                                                                         │
│       ├── internally manages QueryObserver for query 1                          │
│       ├── internally manages QueryObserver for query 2                          │
│       └── internally manages QueryObserver for query 3                          │
│                                                                                 │
│  Benefit: When ANY query changes, you get ALL results in one callback           │
│           No manual coordination, no race conditions                            │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 6.4 Parallel Execution Timeline

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  this.queryMultiple({                                                           │
│    queries: [                                                                   │
│      { queryKey: ['users'], queryFn: fetchUsers },      // 300ms               │
│      { queryKey: ['posts'], queryFn: fetchPosts },      // 100ms               │
│      { queryKey: ['settings'], queryFn: fetchSettings } // 200ms               │
│    ],                                                                           │
│    onStateChange: (results) => { ... }                                          │
│  })                                                                             │
│                                                                                 │
│  Timeline:                                                                      │
│  ═════════                                                                      │
│                                                                                 │
│  t=0ms     ┌─────────────────────────────────────────────────────────────────┐  │
│            │ ALL 3 HTTP REQUESTS START SIMULTANEOUSLY                        │  │
│            │                                                                 │  │
│            │   GET /users    ════════════════════════════════════►           │  │
│            │   GET /posts    ══════════►                                     │  │
│            │   GET /settings ════════════════►                               │  │
│            └─────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│            onStateChange([                                                      │
│              { status: 'pending', data: undefined },                            │
│              { status: 'pending', data: undefined },                            │
│              { status: 'pending', data: undefined },                            │
│            ])                                                                   │
│                                                                                 │
│  t=100ms   posts completes first                                                │
│            onStateChange([                                                      │
│              { status: 'pending', data: undefined },                            │
│              { status: 'success', data: [...posts] },  ◄── UPDATED             │
│              { status: 'pending', data: undefined },                            │
│            ])                                                                   │
│                                                                                 │
│  t=200ms   settings completes                                                   │
│            onStateChange([                                                      │
│              { status: 'pending', data: undefined },                            │
│              { status: 'success', data: [...posts] },                           │
│              { status: 'success', data: {...settings} }, ◄── UPDATED           │
│            ])                                                                   │
│                                                                                 │
│  t=300ms   users completes last                                                 │
│            onStateChange([                                                      │
│              { status: 'success', data: [...users] },   ◄── UPDATED            │
│              { status: 'success', data: [...posts] },                           │
│              { status: 'success', data: {...settings} },                        │
│            ])                                                                   │
│                                                                                 │
│  ═══════════════════════════════════════════════════════════════════════════    │
│                                                                                 │
│  TOTAL TIME: 300ms (slowest query)                                              │
│  NOT: 300 + 100 + 200 = 600ms (if sequential)                                   │
│                                                                                 │
│  KEY POINT: onStateChange fires MULTIPLE TIMES (once per query completion)      │
│             Each invocation receives ALL results as an array                    │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 6.5 Dynamic Queries (Like useQueries with .map())

From the course:
> "Perhaps the best part about the useQueries hook is that the array of queries you pass to it can be dynamic."

```typescript
// Fetch issues for ALL repos (dynamic number of queries)
await this.queryMultiple(() => ({
  queries: this.getStateValue().repos.map(repo => ({
    queryKey: ['repos', repo.name, 'issues'],
    queryFn: () => fetchIssues(repo.name),
  })),
  combine: (results) => ({
    allIssues: results.flatMap(r => r.data ?? []),
    totalCount: results.reduce((sum, r) => sum + (r.data?.length ?? 0), 0),
    isLoading: results.some(r => r.status === 'pending'),
  }),
  onStateChange: (combined) => {
    this.updateState(combined)
  }
}))
```

When `repos` changes (e.g., new repo added), `onStateChange()` on BaseViewModel triggers, which calls `observer.setQueries()` with the new array. QueriesObserver handles adding/removing queries atomically.

---

## 7. Type System Design

### 7.1 Matching React Query's Type Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  REACT QUERY TYPE HIERARCHY              OUR MVVM TYPE HIERARCHY                │
│  ═══════════════════════════             ══════════════════════════             │
│                                                                                 │
│  QueryObserverOptions<                   QueryObserverOptions<                  │
│    TQueryFnData,                           TQueryFnData,                        │
│    TError,                                 TError,                              │
│    TData,                                  TData,                               │
│    TQueryData,                             TQueryData,                          │
│    TQueryKey extends QueryKey              TQueryKey extends QueryKey  ✅       │
│  >                                       >                                      │
│       │                                       │                                 │
│       │ extends                               │ extends                         │
│       ▼                                       ▼                                 │
│  UseBaseQueryOptions                     UseBaseQueryOptions                    │
│    + subscribed?: boolean                  + onStateChange?: callback  ✅       │
│       │                                       │                                 │
│       │ extends (omits 'suspense')            │ extends (omits 'suspense')      │
│       ▼                                       ▼                                 │
│  UseQueryOptions<                        QueryOptions<                          │
│    TQueryFnData,                           TQueryFnData,                        │
│    TError,                                 TError,                              │
│    TData,                                  TData,                               │
│    TQueryKey                               TQueryKey                   ✅       │
│  >                                       >                                      │
│       │                                       │                                 │
│       │ used by                               │ used by                         │
│       ▼                                       ▼                                 │
│  queryOptions()                          queryOptions()                         │
│    → DataTag<TQueryKey, ...>               → DataTag<TQueryKey, ...>   ✅       │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Why DataTag Matters

```typescript
// WITHOUT DataTag
const userQuery = {
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId),
}

// Cache access requires MANUAL type
const user = this.getQueryData<User>(['user', userId])
//                             ^^^^^^ Manual! Can be wrong!

// ═══════════════════════════════════════════════════════════════════════════════

// WITH DataTag (via queryOptions helper)
const userQuery = queryOptions({
  queryKey: ['user', userId] as const,
  queryFn: () => fetchUser(userId),
})

// Cache access is AUTOMATIC
const user = this.getQueryData(userQuery.queryKey)
//    ^? User | undefined  ← INFERRED! No manual annotation!
```

### 7.3 The queryOptions() Overloads

Three overloads handle different initialData scenarios:

1. **DefinedInitialDataOptions** - When initialData guarantees data exists
2. **UnusedSkipTokenOptions** - When queryFn doesn't use SkipToken
3. **UndefinedInitialDataOptions** - Default case

This matches React Query exactly, ensuring type-safe factories.

---

## Summary

### Key Architecture Decisions

| Decision | Why |
|----------|-----|
| One observer per query() | Deduplication happens at Query level, not Observer level |
| QueriesObserver for queryMultiple() | Single subscription, atomic updates, combined results |
| Always use defaultQueryOptions() | Gets all configured behaviors |
| No custom hashing | TanStack handles comparison internally |
| getOptimisticResult() for initial state | Handles placeholderData, initialData, etc. |
| notifyManager.batchCalls() | Prevents multiple rapid callbacks |
| mount() at app startup | Enables refetchOnWindowFocus, refetchOnReconnect |
| Function-based options | Enables dynamic queries that re-evaluate on state change |
| onStateChange callback | Bridges TanStack's PULL model to MVVM's PUSH model |
| DataTag in queryOptions() | Enables type-safe cache access |

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
| Parallel Queries | Via QueriesObserver |
| TypeScript | Full inference via DataTag + queryOptions() |

---

*For implementation details, see: `tanstack-query-mvvm-implementation.md`*
