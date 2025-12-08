# TanStack Query MVVM Adapter v4: Final Architecture

## Version History

| Version | Changes |
|---------|---------|
| v1 | Initial design with shared observers (buggy) |
| v2 | Added queryOptions pattern, onStateChange explanation |
| v3 | Observer-per-usage, QueryHandle class, custom hashing |
| **v4** | **Removed QueryHandle, removed custom hashing, matches React/MobX exactly** |

---

## What Changed from v3 → v4

### Change 1: Removed QueryHandle Class

**v3 (Wrong):**
```typescript
// Separate class with methods
class QueryHandle {
  private observer: QueryObserver
  private currentHash: string
  
  start() { /* ... */ }
  checkAndUpdate() { /* ... */ }
  private hashOptions() { /* ... */ }
}
```

**v4 (Correct):**
```typescript
// Simple interface - just data, no methods
interface QueryEntry {
  getOptions: () => QueryObserverOptions
  observer: QueryObserver
  unsubscribe: () => void
}
```

**Why:** Neither React nor MobX have a "QueryHandle" abstraction. They just store the observer and call methods on it directly. Adding a class was unnecessary ceremony.

---

### Change 2: Removed Custom Options Hashing

**v3 (Wrong):**
```typescript
// We were manually hashing and comparing options
private hashOptions(options): string {
  return JSON.stringify({
    queryKey: options.queryKey,
    enabled: options.enabled,
    staleTime: options.staleTime,
  })
}

checkAndUpdate(): void {
  const newHash = this.hashOptions(newOptions)
  if (newHash !== this.currentHash) {  // Manual comparison
    this.currentHash = newHash
    this.observer.setOptions(newOptions)
  }
}
```

**v4 (Correct):**
```typescript
// Just call setOptions - TanStack handles comparison internally
onStateChange(): void {
  for (const entry of this.queries) {
    const options = entry.getOptions()
    const defaulted = client.defaultQueryOptions(options)
    entry.observer.setOptions(defaulted)  // That's it!
  }
}
```

**Why:** This is exactly what React and MobX do. They don't hash or compare options themselves - they trust TanStack's internal optimization. From React's `useBaseQuery`:

```typescript
// React Query - NO hashing, just call setOptions every render
React.useEffect(() => {
  observer.setOptions(defaultedOptions)
}, [defaultedOptions, observer])
```

And from MobX-TanStack-Query:

```typescript
// MobX - NO hashing, just call setOptions
update(optionsUpdate) {
  this.options = { ...this.options, ...optionsUpdate }
  this.queryObserver.setOptions(this.options)  // That's it!
}
```

---

### Change 3: Simplified QueryCapability

**v3:** ~200 lines with QueryHandle class
**v4:** ~100 lines, just QueryCapability with inline logic

**Why:** Less code, same behavior, easier to understand and maintain.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  ViewModel                                                                      │
│  ─────────                                                                      │
│                                                                                 │
│    this.query(() => ({                                                          │
│      queryKey: ['voices', this.getStateValue().page],                           │
│      queryFn: () => fetchVoices(this.getStateValue().page),                     │
│      enabled: this.getStateValue().isReady,                                     │
│      onStateChange: (result) => this.updateState({ voices: result.data })       │
│    }))                                                                          │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  BaseViewModel                                                          │    │
│  │                                                                         │    │
│  │  • Owns state$: BehaviorSubject<TState>                                 │    │
│  │  • On state change → calls queryCapability.onStateChange()              │    │
│  │  • Exposes this.query() to child ViewModels                             │    │
│  │                                                                         │    │
│  │  Code added: ~10 lines                                                  │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  QueryCapability                                                        │    │
│  │                                                                         │    │
│  │  • Stores queries: QueryEntry[]                                         │    │
│  │  • fetch() → creates observer, subscribes, stores entry                 │    │
│  │  • onStateChange() → loops entries, calls setOptions() on each          │    │
│  │                                                                         │    │
│  │  NO QueryHandle class. NO custom hashing.                               │    │
│  │  Just store observer, call setOptions. TanStack handles the rest.       │    │
│  │                                                                         │    │
│  │  Code: ~100 lines                                                       │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│         │                                                                       │
│         ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  @tanstack/query-core                                                   │    │
│  │                                                                         │    │
│  │  QueryObserver.setOptions() internally:                                 │    │
│  │  • Compares new options with current                                    │    │
│  │  • If queryKey changed → switches to different Query in cache           │    │
│  │  • If enabled changed → starts/stops fetching                           │    │
│  │  • If staleTime changed → updates staleness calculation                 │    │
│  │  • Triggers fetch if needed                                             │    │
│  │  • Notifies subscribers                                                 │    │
│  │                                                                         │    │
│  │  WE DON'T REIMPLEMENT THIS. We just call setOptions().                  │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## The Core Insight

### React's Mental Model

```typescript
function VoicesScreen() {
  const [page, setPage] = useState(1)
  
  // On EVERY render, this runs with current state
  const { data } = useQuery({
    queryKey: ['voices', page],
    queryFn: () => fetchVoices(page),
    enabled: page > 0,
  })
  
  return <Button onPress={() => setPage(2)}>Next</Button>
}
```

When `setPage(2)` is called:
1. Component re-renders
2. `useQuery` runs again with `page = 2`
3. Internally, `observer.setOptions()` is called with new options
4. TanStack figures out what changed and acts accordingly

### Our MVVM Mental Model

```typescript
class VoicesViewModel extends BaseViewModel<State> {
  initialize() {
    // Register a query with a function that reads current state
    this.query(() => {
      const { page } = this.getStateValue()
      return {
        queryKey: ['voices', page],
        queryFn: () => fetchVoices(page),
        enabled: page > 0,
        onStateChange: (result) => this.updateState({ voices: result.data })
      }
    })
  }
  
  setPage(page: number) {
    this.updateState({ page: 2 })
  }
}
```

When `setPage(2)` is called:
1. `state$` emits new state
2. `queryCapability.onStateChange()` is called
3. For each query, `getOptions()` runs with current state
4. `observer.setOptions()` is called with new options
5. TanStack figures out what changed and acts accordingly

**Same flow. Same behavior. Just different trigger mechanism.**

---

## Implementation

### File Structure

```
core/
├── query/
│   ├── index.ts              # Public exports
│   ├── queryClient.ts        # Singleton management (~50 lines)
│   ├── QueryCapability.ts    # Core adapter (~100 lines)
│   └── types.ts              # Type definitions (~30 lines)
│
└── viewmodels/
    └── BaseViewModel.ts      # Integration (~15 lines added)
```

### types.ts

```typescript
import type {
  QueryKey,
  QueryObserverOptions,
  QueryObserverResult,
  MutationObserverOptions,
  MutationObserverResult,
} from '@tanstack/query-core'

/**
 * Query options with our onStateChange callback.
 * Extends TanStack's QueryObserverOptions.
 */
export interface QueryOptions<
  TQueryFnData = unknown,
  TError = Error,
  TData = TQueryFnData,
> extends Omit<QueryObserverOptions<TQueryFnData, TError, TData, TQueryFnData, QueryKey>, 'queryKey'> {
  queryKey: QueryKey
  
  /**
   * Called on EVERY state change:
   * - Immediately with optimistic/cached result
   * - When fetch starts (isFetching: true)
   * - When fetch completes (success or error)
   * - On background refetch
   * - On invalidation from another ViewModel
   * - On window focus refetch
   * - On network reconnect refetch
   */
  onStateChange?: (result: QueryObserverResult<TData, TError>) => void
}

/**
 * Options can be a static object OR a function returning options.
 * 
 * Static: Query doesn't depend on ViewModel state
 * Function: Query re-evaluates when state changes (like React re-render)
 */
export type QueryOptionsOrGetter<
  TQueryFnData = unknown,
  TError = Error,
  TData = TQueryFnData,
> = 
  | QueryOptions<TQueryFnData, TError, TData>
  | (() => QueryOptions<TQueryFnData, TError, TData>)

/**
 * Mutation options with our onStateChange callback.
 */
export interface MutationOptions<
  TData = unknown,
  TError = Error,
  TVariables = void,
  TContext = unknown,
> extends MutationObserverOptions<TData, TError, TVariables, TContext> {
  onStateChange?: (result: MutationObserverResult<TData, TError, TVariables, TContext>) => void
}

/**
 * Handle returned by this.mutation()
 */
export interface MutationHandle<TData, TError, TVariables> {
  mutate: (variables: TVariables) => void
  mutateAsync: (variables: TVariables) => Promise<TData>
  getState: () => MutationObserverResult<TData, TError, TVariables>
  reset: () => void
}

// Re-export TanStack types for convenience
export type {
  QueryKey,
  QueryObserverResult,
  MutationObserverResult,
} from '@tanstack/query-core'
```

### queryClient.ts

```typescript
import { QueryClient, type QueryClientConfig } from '@tanstack/query-core'

let queryClient: QueryClient | null = null

/**
 * Initialize the QueryClient singleton.
 * Call ONCE at app startup, before any ViewModel is created.
 */
export function initializeQueryClient(config?: QueryClientConfig): QueryClient {
  if (queryClient) {
    console.warn('QueryClient already initialized')
    return queryClient
  }
  
  queryClient = new QueryClient(config)
  
  // CRITICAL: Mount enables refetchOnWindowFocus, refetchOnReconnect
  queryClient.mount()
  
  return queryClient
}

/**
 * Get the QueryClient singleton.
 */
export function getQueryClient(): QueryClient {
  if (!queryClient) {
    throw new Error(
      'QueryClient not initialized. Call initializeQueryClient() at app startup.'
    )
  }
  return queryClient
}

/**
 * Clear all cached data. Call on logout.
 */
export function clearQueryCache(): void {
  queryClient?.clear()
}

/**
 * Destroy the QueryClient. Call on app teardown (if ever).
 */
export function destroyQueryClient(): void {
  if (queryClient) {
    queryClient.unmount()
    queryClient = null
  }
}
```

### QueryCapability.ts

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
 * Manages QueryObservers and MutationObservers for a ViewModel.
 * Does NOT know about RxJS or BehaviorSubject - just TanStack Query.
 */
export class QueryCapability {
  private queries: QueryEntry[] = []
  private mutations: MutationEntry[] = []
  private isDestroyed = false

  /**
   * Create and start a query.
   * 
   * @param optionsOrGetter - Static options object OR function returning options.
   *   - Static: `{ queryKey: [...], queryFn: ... }`
   *   - Dynamic: `() => ({ queryKey: [...], queryFn: ... })`
   * 
   * When options is a function, it's re-evaluated on every state change
   * (via onStateChange), just like React re-evaluates useQuery on every render.
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
    
    // Get initial options and apply defaults
    const options = getOptions()
    const defaultedOptions = client.defaultQueryOptions(
      options as QueryObserverOptions
    )
    defaultedOptions._optimisticResults = 'optimistic'
    
    // Create observer ONCE (same as React's useState(() => new Observer(...)))
    const observer = new QueryObserver(client, defaultedOptions)
    
    // Prime UI with optimistic result immediately
    const optimisticResult = observer.getOptimisticResult(defaultedOptions)
    options.onStateChange?.(optimisticResult as QueryObserverResult<TData, TError>)
    
    // Subscribe for future updates
    const unsubscribe = observer.subscribe(
      notifyManager.batchCalls(() => {
        if (this.isDestroyed) return
        
        // Re-get options to get current onStateChange callback
        // (the callback itself might have changed if options is a function)
        const currentOptions = getOptions()
        currentOptions.onStateChange?.(
          observer.getCurrentResult() as QueryObserverResult<TData, TError>
        )
      })
    )
    
    // Catch updates between observer creation and subscription
    observer.updateResult()
    
    // Store entry
    this.queries.push({ getOptions, observer, unsubscribe })
    
    // Return data for imperative usage
    const result = observer.getCurrentResult()
    
    if (result.status === 'success' && !result.isStale) {
      return result.data as TData
    }
    
    if (defaultedOptions.enabled === false) {
      return result.data as TData | undefined
    }
    
    try {
      const fetchResult = await observer.refetch()
      return fetchResult.data as TData | undefined
    } catch {
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
   */
  onStateChange(): void {
    if (this.isDestroyed) return
    
    const client = getQueryClient()
    
    for (const entry of this.queries) {
      // Re-evaluate options (reads current state via closure)
      const options = entry.getOptions()
      const defaultedOptions = client.defaultQueryOptions(
        options as QueryObserverOptions
      )
      defaultedOptions._optimisticResults = 'optimistic'
      
      // Just call setOptions - TanStack handles comparison internally
      // This is EXACTLY what React's useBaseQuery does
      entry.observer.setOptions(defaultedOptions)
    }
  }

  /**
   * Create a mutation.
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
              observer.getCurrentResult() as MutationObserverResult<TData, TError, TVariables, TContext>
            )
          }
        })
      )
    }
    
    this.mutations.push({ observer, unsubscribe })
    
    return {
      mutate: (variables: TVariables) => observer.mutate(variables),
      mutateAsync: (variables: TVariables) => observer.mutateAsync(variables),
      getState: () => observer.getCurrentResult() as MutationObserverResult<TData, TError, TVariables>,
      reset: () => observer.reset(),
    }
  }

  /**
   * Prefetch data (no observer, just cache warming).
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
   */
  getData<TData = unknown>(queryKey: QueryKey): TData | undefined {
    return getQueryClient().getQueryData(queryKey)
  }

  /**
   * Set query data directly (for optimistic updates).
   */
  setData<TData = unknown>(
    queryKey: QueryKey,
    updater: TData | ((old: TData | undefined) => TData)
  ): TData | undefined {
    return getQueryClient().setQueryData(queryKey, updater)
  }

  /**
   * Invalidate queries to trigger refetch.
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
   */
  async cancel(filters: { queryKey?: QueryKey } = {}): Promise<void> {
    await getQueryClient().cancelQueries(filters)
  }

  /**
   * Clean up all observers and subscriptions.
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
      throw new Error('QueryCapability has been destroyed')
    }
  }
}
```

### BaseViewModel.ts

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

export abstract class BaseViewModel<TState> {
  protected state$: BehaviorSubject<TState>
  
  private __queryCapability = new QueryCapability()
  private __stateSubscription: Subscription
  
  constructor(initialState: TState) {
    this.state$ = new BehaviorSubject(initialState)
    
    // When state changes, notify QueryCapability to re-evaluate queries
    // This is equivalent to React component re-rendering
    this.__stateSubscription = this.state$.pipe(
      skip(1)  // Skip initial value
    ).subscribe(() => {
      this.__queryCapability.onStateChange()
    })
  }
  
  protected getStateValue(): TState {
    return this.state$.getValue()
  }
  
  protected updateState(partial: Partial<TState>): void {
    this.state$.next({ ...this.state$.getValue(), ...partial })
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // Query API
  // ═══════════════════════════════════════════════════════════════════════════
  
  /**
   * Fetch data with TanStack Query.
   * 
   * Static query (doesn't depend on state):
   * ```typescript
   * await this.query({
   *   queryKey: ['settings'],
   *   queryFn: fetchSettings,
   *   onStateChange: (result) => this.updateState({ settings: result.data })
   * })
   * ```
   * 
   * Dynamic query (re-evaluates when state changes):
   * ```typescript
   * await this.query(() => ({
   *   queryKey: ['voices', this.getStateValue().page],
   *   queryFn: () => fetchVoices(this.getStateValue().page),
   *   enabled: this.getStateValue().isReady,
   *   onStateChange: (result) => this.updateState({ voices: result.data })
   * }))
   * ```
   */
  protected query<TQueryFnData = unknown, TError = Error, TData = TQueryFnData>(
    optionsOrGetter: QueryOptionsOrGetter<TQueryFnData, TError, TData>
  ): Promise<TData | undefined> {
    return this.__queryCapability.fetch(optionsOrGetter)
  }
  
  /**
   * Create a mutation.
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
   */
  protected prefetchQuery<TData = unknown>(options: {
    queryKey: QueryKey
    queryFn: () => Promise<TData>
    staleTime?: number
  }): Promise<void> {
    return this.__queryCapability.prefetch(options)
  }
  
  /**
   * Get cached data synchronously.
   */
  protected getQueryData<TData = unknown>(queryKey: QueryKey): TData | undefined {
    return this.__queryCapability.getData(queryKey)
  }
  
  /**
   * Set query data directly (for optimistic updates).
   */
  protected setQueryData<TData = unknown>(
    queryKey: QueryKey,
    updater: TData | ((old: TData | undefined) => TData)
  ): TData | undefined {
    return this.__queryCapability.setData(queryKey, updater)
  }
  
  /**
   * Invalidate queries to trigger refetch.
   */
  protected invalidateQueries(filters?: {
    queryKey?: QueryKey
    exact?: boolean
    refetchType?: 'active' | 'inactive' | 'all' | 'none'
  }): Promise<void> {
    return this.__queryCapability.invalidate(filters)
  }
  
  /**
   * Cancel outgoing queries.
   */
  protected cancelQueries(filters?: { queryKey?: QueryKey }): Promise<void> {
    return this.__queryCapability.cancel(filters)
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // Lifecycle
  // ═══════════════════════════════════════════════════════════════════════════
  
  destroy(): void {
    this.__stateSubscription.unsubscribe()
    this.__queryCapability.destroy()
    this.state$.complete()
  }
}
```

---

## How It Works: Complete Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  t=0  ViewModel.initialize() runs                                               │
│       │                                                                         │
│       └──► this.query(() => ({                                                  │
│              queryKey: ['voices', this.getStateValue().page],  // page = 1      │
│              queryFn: () => fetchVoices(this.getStateValue().page),             │
│              enabled: this.getStateValue().isReady,            // true          │
│              onStateChange: (result) => this.updateState({ voices: result.data })│
│            }))                                                                  │
│                 │                                                               │
│                 ▼                                                               │
│       QueryCapability.fetch(optionsOrGetter):                                   │
│       │                                                                         │
│       ├──► getOptions = optionsOrGetter  (it's a function)                      │
│       │                                                                         │
│       ├──► options = getOptions()  // Evaluates with current state              │
│       │    // { queryKey: ['voices', 1], enabled: true, ... }                   │
│       │                                                                         │
│       ├──► defaultedOptions = client.defaultQueryOptions(options)               │
│       │                                                                         │
│       ├──► observer = new QueryObserver(client, defaultedOptions)               │
│       │                                                                         │
│       ├──► optimisticResult = observer.getOptimisticResult(defaultedOptions)    │
│       ├──► options.onStateChange(optimisticResult)  // UI gets loading state    │
│       │                                                                         │
│       ├──► unsubscribe = observer.subscribe(callback)                           │
│       ├──► observer.updateResult()                                              │
│       │                                                                         │
│       └──► this.queries.push({ getOptions, observer, unsubscribe })             │
│                                                                                 │
│       Observer fetches data → onStateChange called → UI shows voices            │
│                                                                                 │
│  ═══════════════════════════════════════════════════════════════════════════    │
│                                                                                 │
│  t=1  User clicks "Page 2"                                                      │
│       │                                                                         │
│       └──► this.updateState({ page: 2 })                                        │
│                 │                                                               │
│                 ▼                                                               │
│       BehaviorSubject.next({ ...state, page: 2 })                               │
│                 │                                                               │
│                 ▼                                                               │
│       BaseViewModel's state$ subscription fires                                 │
│                 │                                                               │
│                 ▼                                                               │
│       queryCapability.onStateChange()                                           │
│                 │                                                               │
│                 ▼                                                               │
│       For each query in this.queries:                                           │
│       │                                                                         │
│       ├──► options = entry.getOptions()  // Re-evaluates with NEW state!        │
│       │    // { queryKey: ['voices', 2], enabled: true, ... }                   │
│       │                                                                         │
│       ├──► defaultedOptions = client.defaultQueryOptions(options)               │
│       │                                                                         │
│       └──► entry.observer.setOptions(defaultedOptions)                          │
│                 │                                                               │
│                 ▼                                                               │
│       TanStack internally:                                                      │
│       • Sees queryKey changed: ['voices', 1] → ['voices', 2]                    │
│       • Looks up Query for ['voices', 2] in cache (or creates it)               │
│       • Subscribes observer to new Query                                        │
│       • Triggers fetch for page 2                                               │
│       • Calls our subscription callback with new result                         │
│                 │                                                               │
│                 ▼                                                               │
│       onStateChange({ data: [...page2Voices], isFetching: false })              │
│       ViewModel.updateState({ voices: [...page2Voices] })                       │
│       UI shows page 2 voices                                                    │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Comparison: v4 vs React vs MobX

| Aspect | React useQuery | MobX-TanStack-Query | Our v4 |
|--------|---------------|---------------------|--------|
| **Trigger** | Component re-render | MobX reaction | state$ emit |
| **Options evaluation** | Every render | When observed values change | Every state change |
| **setOptions call** | Every render (useEffect) | In update() method | In onStateChange() |
| **Custom hashing** | ❌ None | ❌ None | ❌ None (removed!) |
| **Observer storage** | useState | Class property | QueryEntry array |
| **QueryHandle class** | ❌ No | ❌ No | ❌ No (removed!) |

**We now match React and MobX exactly.**

---

## Usage Examples

### Static Query (Options Object)

```typescript
class SettingsViewModel extends BaseViewModel<SettingsState> {
  async initialize() {
    // Options is an object - never re-evaluated
    await this.query({
      queryKey: ['settings'],
      queryFn: SettingsAPI.fetch,
      staleTime: 10 * 60 * 1000,
      onStateChange: (result) => {
        this.updateState({
          settings: result.data,
          isLoading: result.isLoading,
        })
      }
    })
  }
}
```

### Dynamic Query (Options Function)

```typescript
class VoicesViewModel extends BaseViewModel<VoicesState> {
  async initialize() {
    // Options is a function - re-evaluated on every state change
    await this.query(() => {
      const { page, search, isReady, isPriority } = this.getStateValue()
      
      return {
        queryKey: ['voices', { page, search }],
        queryFn: () => VoiceAPI.getVoices({ page, search }),
        
        // ALL of these can be dynamic!
        enabled: isReady,
        staleTime: isPriority ? 0 : 5 * 60 * 1000,
        refetchInterval: page === 1 ? 30000 : false,
        
        onStateChange: (result) => {
          this.updateState({
            voices: result.data ?? [],
            isFetching: result.isFetching,
            error: result.error?.message ?? null,
          })
        }
      }
    })
  }
  
  setPage(page: number) {
    this.updateState({ page })
    // Query automatically re-evaluates and fetches page 2!
  }
  
  setSearch(search: string) {
    this.updateState({ search })
    // Query automatically re-evaluates with new search!
  }
  
  setPriority(isPriority: boolean) {
    this.updateState({ isPriority })
    // staleTime changes from 5min to 0!
  }
}
```

### Mutation with Optimistic Update

```typescript
class VoiceEditorViewModel extends BaseViewModel<EditorState> {
  private updateMutation!: MutationHandle<Voice, Error, UpdateVoiceInput>
  
  async initialize(voiceId: string) {
    this.setupMutations()
    await this.fetchVoice(voiceId)
  }
  
  private setupMutations() {
    this.updateMutation = this.mutation({
      mutationFn: (input: UpdateVoiceInput) =>
        VoiceAPI.update(input.id, input.data),
      
      onMutate: async (input) => {
        await this.cancelQueries({ queryKey: ['voices'] })
        
        const previous = this.getQueryData<Voice>(['voice', input.id])
        
        this.setQueryData<Voice>(['voice', input.id], (old) =>
          old ? { ...old, ...input.data } : old
        )
        
        return { previous }
      },
      
      onSuccess: () => {
        this.invalidateQueries({ queryKey: ['voices'] })
      },
      
      onError: (error, input, context) => {
        if (context?.previous) {
          this.setQueryData(['voice', input.id], context.previous)
        }
        this.updateState({ error: error.message })
      },
      
      onStateChange: (result) => {
        this.updateState({ isSaving: result.isPending })
      }
    })
  }
  
  saveVoice(data: Partial<Voice>) {
    this.updateMutation.mutate({ id: this.voiceId, data })
  }
}
```

---

## Performance

### Cost Per State Change

```
For each query:
1. getOptions() call          ~1μs   (function call, reads state)
2. defaultQueryOptions()      ~5μs   (merges defaults)
3. observer.setOptions()      ~10-50μs (TanStack comparison + action)

Total: ~15-60μs per query per state change
```

### Realistic Scenario

```
5 queries × 60μs × 10 state changes/second = 3ms/second = 0.3% CPU
```

**Negligible.** This is the same overhead React has on every re-render.

### Why Support Both Static and Dynamic?

**Not for performance** (it's negligible). For **clarity and intent**:

```typescript
// Clearly signals: "This query is static, won't change"
this.query({
  queryKey: ['settings'],
  queryFn: fetchSettings,
})

// Clearly signals: "This query depends on state"
this.query(() => ({
  queryKey: ['voices', this.getStateValue().page],
  queryFn: () => fetchVoices(this.getStateValue().page),
}))
```

Self-documenting code is valuable.

---

## React Native Configuration

```typescript
// app/querySetup.ts
import { AppState } from 'react-native'
import NetInfo from '@react-native-community/netinfo'
import { focusManager, onlineManager } from '@tanstack/query-core'
import { initializeQueryClient } from './core/query/queryClient'

export function setupQueryClient() {
  // Configure for React Native
  focusManager.setEventListener((handleFocus) => {
    const subscription = AppState.addEventListener('change', (state) => {
      handleFocus(state === 'active')
    })
    return () => subscription.remove()
  })
  
  onlineManager.setEventListener((setOnline) => {
    return NetInfo.addEventListener((state) => {
      setOnline(!!state.isConnected)
    })
  })
  
  // Initialize
  initializeQueryClient({
    defaultOptions: {
      queries: {
        staleTime: 1000 * 60,
        gcTime: 1000 * 60 * 5,
        retry: 3,
        refetchOnWindowFocus: true,
        refetchOnReconnect: true,
      }
    }
  })
}

// Call at app startup
setupQueryClient()
```

---

## Summary

### What v4 Removes (Unnecessary Complexity)

- ❌ QueryHandle class
- ❌ Custom options hashing
- ❌ isDynamic flag checking
- ❌ Manual comparison logic

### What v4 Keeps (Essential)

- ✅ QueryEntry interface (just data, no methods)
- ✅ Observer-per-query pattern
- ✅ setOptions() on state change
- ✅ Static vs dynamic options (for clarity)
- ✅ All TanStack features (caching, deduplication, background refetch, etc.)

### The Key Insight

**We don't need to be smart. TanStack is already smart.**

React and MobX both just call `observer.setOptions()` and let TanStack handle the comparison. We should do the same.

---

## Code Stats

| File | Lines | Purpose |
|------|-------|---------|
| types.ts | ~50 | Type definitions |
| queryClient.ts | ~50 | Singleton management |
| QueryCapability.ts | ~100 | Core adapter |
| BaseViewModel.ts | +15 | Integration |
| **Total new code** | **~215** | Full TanStack Query integration |

---

*v4 - Simplified architecture matching React and MobX exactly*
