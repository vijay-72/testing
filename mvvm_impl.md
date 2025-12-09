# TanStack Query MVVM Adapter: Implementation Guide

## Document Purpose

This document contains the **complete, production-ready implementation** of the TanStack Query MVVM adapter. All code here is ready to copy and use.

For architecture explanations and design decisions, see: `tanstack-query-mvvm-architecture.md`

---

## Table of Contents

1. [File Structure](#1-file-structure)
2. [types.ts - Complete Type Definitions](#2-typests---complete-type-definitions)
3. [queryClient.ts - Singleton Management](#3-queryclientts---singleton-management)
4. [QueryCapability.ts - Core Adapter](#4-querycapabilityts---core-adapter)
5. [BaseViewModel.ts - Integration](#5-baseviewmodelts---integration)
6. [Query Factories Pattern](#6-query-factories-pattern)
7. [Usage Examples](#7-usage-examples)
8. [Platform Configuration](#8-platform-configuration)
9. [Testing Strategy](#9-testing-strategy)

---

## 1. File Structure

```
core/
├── query/
│   ├── index.ts              # Public exports
│   ├── types.ts              # Type definitions (~150 lines)
│   ├── queryClient.ts        # Singleton management (~60 lines)
│   └── QueryCapability.ts    # Core adapter (~300 lines)
│
├── viewmodels/
│   └── BaseViewModel.ts      # Integration (~50 lines added)
│
└── queries/                  # Query factories (per domain)
    ├── voiceQueries.ts
    ├── settingsQueries.ts
    └── userQueries.ts
```

---

## 2. types.ts - Complete Type Definitions

This file matches React Query's type system exactly, with our MVVM-specific `onStateChange` callback added.

```typescript
// types.ts
// Aligned with React Query's type system for full compatibility

import type {
  QueryKey,
  QueryObserverOptions,
  QueryObserverResult,
  MutationObserverOptions,
  MutationObserverResult,
  DefaultError,
  OmitKeyof,
  DataTag,
  InitialDataFunction,
  NonUndefinedGuard,
  QueryFunction,
  SkipToken,
  WithRequired,
} from '@tanstack/query-core'

// ═══════════════════════════════════════════════════════════════════════════════
// BASE QUERY OPTIONS
// Matches React Query's UseBaseQueryOptions
// ═══════════════════════════════════════════════════════════════════════════════

/**
 * Base query options - extends QueryObserverOptions from core.
 * 
 * This matches React Query's UseBaseQueryOptions structure.
 * We add our MVVM-specific `onStateChange` callback.
 * 
 * @template TQueryFnData - Type returned by queryFn
 * @template TError - Error type (defaults to DefaultError)
 * @template TData - Type after select() transform (defaults to TQueryFnData)
 * @template TQueryData - Type stored in cache (defaults to TQueryFnData)
 * @template TQueryKey - Exact query key type (preserves literal types)
 */
export interface UseBaseQueryOptions<
  TQueryFnData = unknown,
  TError = DefaultError,
  TData = TQueryFnData,
  TQueryData = TQueryFnData,
  TQueryKey extends QueryKey = QueryKey,
> extends QueryObserverOptions<
    TQueryFnData,
    TError,
    TData,
    TQueryData,
    TQueryKey
  > {
  /**
   * Called on EVERY query state change.
   * This is the MVVM equivalent of React re-rendering with new data.
   * 
   * Fires when:
   * - Fetch starts (status: 'pending', isFetching: true)
   * - Fetch completes (status: 'success' or 'error')
   * - Background refetch starts/completes
   * - Window focus refetch
   * - Network reconnect refetch
   * - Manual invalidation
   * - Retry attempts
   * 
   * @example
   * onStateChange: (result) => {
   *   this.updateState({
   *     data: result.data ?? [],
   *     isLoading: result.isLoading,
   *     isFetching: result.isFetching,
   *     error: result.error?.message ?? null,
   *   })
   * }
   */
  onStateChange?: (result: QueryObserverResult<TData, TError>) => void
}

// ═══════════════════════════════════════════════════════════════════════════════
// QUERY OPTIONS
// Matches React Query's UseQueryOptions (omits suspense since we don't use it)
// ═══════════════════════════════════════════════════════════════════════════════

/**
 * Query options for single queries.
 * This is the MVVM equivalent of React Query's UseQueryOptions.
 * 
 * Omits 'suspense' since MVVM doesn't use React Suspense.
 */
export interface QueryOptions<
  TQueryFnData = unknown,
  TError = DefaultError,
  TData = TQueryFnData,
  TQueryKey extends QueryKey = QueryKey,
> extends OmitKeyof<
    UseBaseQueryOptions<TQueryFnData, TError, TData, TQueryFnData, TQueryKey>,
    'suspense'
  > {}

/**
 * Query options can be static or a function (for dynamic queries).
 * 
 * Use a function when query depends on ViewModel state:
 * - Query key includes state values (pagination, filters)
 * - enabled flag depends on state
 * - queryFn needs current state values
 * 
 * @example
 * // Static - query doesn't depend on state
 * this.query({
 *   queryKey: ['settings'],
 *   queryFn: fetchSettings,
 * })
 * 
 * // Dynamic - re-evaluates when state changes
 * this.query(() => ({
 *   queryKey: ['voices', this.getStateValue().page],
 *   queryFn: () => fetchVoices(this.getStateValue().page),
 *   enabled: this.getStateValue().isReady,
 * }))
 */
export type QueryOptionsOrGetter<
  TQueryFnData = unknown,
  TError = DefaultError,
  TData = TQueryFnData,
  TQueryKey extends QueryKey = QueryKey,
> =
  | QueryOptions<TQueryFnData, TError, TData, TQueryKey>
  | (() => QueryOptions<TQueryFnData, TError, TData, TQueryKey>)

// ═══════════════════════════════════════════════════════════════════════════════
// queryOptions() HELPER
// Matches React Query's queryOptions with DataTag for type-safe cache access
// ═══════════════════════════════════════════════════════════════════════════════

/**
 * Options with undefined initialData (default case)
 */
export type UndefinedInitialDataOptions<
  TQueryFnData = unknown,
  TError = DefaultError,
  TData = TQueryFnData,
  TQueryKey extends QueryKey = QueryKey,
> = QueryOptions<TQueryFnData, TError, TData, TQueryKey> & {
  initialData?:
    | undefined
    | InitialDataFunction<NonUndefinedGuard<TQueryFnData>>
    | NonUndefinedGuard<TQueryFnData>
}

/**
 * Options without SkipToken in queryFn
 */
export type UnusedSkipTokenOptions<
  TQueryFnData = unknown,
  TError = DefaultError,
  TData = TQueryFnData,
  TQueryKey extends QueryKey = QueryKey,
> = OmitKeyof<QueryOptions<TQueryFnData, TError, TData, TQueryKey>, 'queryFn'> & {
  queryFn?: Exclude<
    QueryOptions<TQueryFnData, TError, TData, TQueryKey>['queryFn'],
    SkipToken | undefined
  >
}

/**
 * Options with defined initialData (guarantees data is always available)
 */
export type DefinedInitialDataOptions<
  TQueryFnData = unknown,
  TError = DefaultError,
  TData = TQueryFnData,
  TQueryKey extends QueryKey = QueryKey,
> = Omit<QueryOptions<TQueryFnData, TError, TData, TQueryKey>, 'queryFn'> & {
  initialData:
    | NonUndefinedGuard<TQueryFnData>
    | (() => NonUndefinedGuard<TQueryFnData>)
  queryFn?: QueryFunction<TQueryFnData, TQueryKey>
}

/**
 * Type-safe query options factory.
 * 
 * Creates query options with a DataTag on the queryKey, enabling type-safe
 * cache access via getQueryData(), setQueryData(), etc.
 * 
 * THREE OVERLOADS (matching React Query):
 * 1. DefinedInitialDataOptions - When initialData is defined
 * 2. UnusedSkipTokenOptions - When queryFn doesn't use SkipToken
 * 3. UndefinedInitialDataOptions - Default case
 * 
 * @example
 * // Define type-safe query options
 * const voiceQueryOptions = (id: string) => queryOptions({
 *   queryKey: ['voice', id] as const,
 *   queryFn: () => fetchVoice(id),
 *   staleTime: 5 * 60 * 1000,
 * })
 * 
 * // Use in ViewModel
 * await this.query({
 *   ...voiceQueryOptions(voiceId),
 *   onStateChange: (result) => this.updateState({ voice: result.data })
 * })
 * 
 * // Type-safe cache access!
 * const voice = this.getQueryData(voiceQueryOptions(voiceId).queryKey)
 * //    ^? Voice | undefined  ← INFERRED from queryFn return type!
 */

// Overload 1: Defined initialData
export function queryOptions<
  TQueryFnData = unknown,
  TError = DefaultError,
  TData = TQueryFnData,
  TQueryKey extends QueryKey = QueryKey,
>(
  options: DefinedInitialDataOptions<TQueryFnData, TError, TData, TQueryKey>,
): DefinedInitialDataOptions<TQueryFnData, TError, TData, TQueryKey> & {
  queryKey: DataTag<TQueryKey, TQueryFnData, TError>
}

// Overload 2: No SkipToken
export function queryOptions<
  TQueryFnData = unknown,
  TError = DefaultError,
  TData = TQueryFnData,
  TQueryKey extends QueryKey = QueryKey,
>(
  options: UnusedSkipTokenOptions<TQueryFnData, TError, TData, TQueryKey>,
): UnusedSkipTokenOptions<TQueryFnData, TError, TData, TQueryKey> & {
  queryKey: DataTag<TQueryKey, TQueryFnData, TError>
}

// Overload 3: Undefined initialData (default)
export function queryOptions<
  TQueryFnData = unknown,
  TError = DefaultError,
  TData = TQueryFnData,
  TQueryKey extends QueryKey = QueryKey,
>(
  options: UndefinedInitialDataOptions<TQueryFnData, TError, TData, TQueryKey>,
): UndefinedInitialDataOptions<TQueryFnData, TError, TData, TQueryKey> & {
  queryKey: DataTag<TQueryKey, TQueryFnData, TError>
}

// Implementation (identity function - magic is in the types)
export function queryOptions(options: unknown) {
  return options
}

// ═══════════════════════════════════════════════════════════════════════════════
// QUERIES OPTIONS (for queryMultiple / useQueries equivalent)
// ═══════════════════════════════════════════════════════════════════════════════

/**
 * Options for queryMultiple (equivalent to useQueries).
 * 
 * @template TQueries - Array of query options
 * @template TCombinedResult - Return type after combine() is applied
 */
export interface QueriesOptions<
  TQueries extends Array<QueryOptions<any, any, any, any>> = Array<
    QueryOptions<any, any, any, any>
  >,
  TCombinedResult = Array<QueryObserverResult<any, any>>,
> {
  /**
   * Array of query options. Each query runs in PARALLEL.
   * 
   * Can be static or dynamic (changes based on state).
   * When state changes, setQueries() handles adding/removing queries atomically.
   */
  queries: TQueries

  /**
   * Optional function to transform the array of results into a custom shape.
   * 
   * Without combine: onStateChange receives Array<QueryObserverResult>
   * With combine: onStateChange receives whatever combine() returns
   * 
   * @example
   * combine: (results) => ({
   *   users: results[0].data ?? [],
   *   posts: results[1].data ?? [],
   *   isLoading: results.some(r => r.status === 'pending'),
   *   errors: results.filter(r => r.error).map(r => r.error),
   * })
   */
  combine?: (results: Array<QueryObserverResult<any, any>>) => TCombinedResult

  /**
   * Called whenever ANY query's state changes.
   * 
   * IMPORTANT: This fires MULTIPLE TIMES as queries resolve!
   * Each invocation receives ALL results (the complete array or combined result).
   * 
   * Timeline example:
   *   t=0ms:   onStateChange([pending, pending, pending])
   *   t=100ms: onStateChange([pending, SUCCESS, pending])   ← query 2 done
   *   t=200ms: onStateChange([pending, SUCCESS, SUCCESS])   ← query 3 done
   *   t=300ms: onStateChange([SUCCESS, SUCCESS, SUCCESS])   ← query 1 done
   * 
   * @example
   * // Without combine - receives raw array
   * onStateChange: (results) => {
   *   const [usersResult, postsResult] = results
   *   this.updateState({
   *     users: usersResult.data ?? [],
   *     posts: postsResult.data ?? [],
   *     isLoading: results.some(r => r.status === 'pending'),
   *   })
   * }
   * 
   * @example
   * // With combine - receives transformed result
   * onStateChange: ({ users, posts, isLoading }) => {
   *   this.updateState({ users, posts, isLoading })
   * }
   */
  onStateChange?: (result: TCombinedResult) => void
}

/**
 * Options can be static or a function (for dynamic queries).
 * 
 * Use a function when queries depend on ViewModel state:
 * - Number of queries changes (e.g., fetch issues for each repo)
 * - Query options change based on state
 * 
 * @example
 * // Static queries
 * this.queryMultiple({
 *   queries: [query1, query2],
 *   onStateChange: ...
 * })
 * 
 * // Dynamic queries (array size can change!)
 * this.queryMultiple(() => ({
 *   queries: this.getStateValue().userIds.map(id => ({
 *     queryKey: ['user', id],
 *     queryFn: () => fetchUser(id),
 *   })),
 *   onStateChange: ...
 * }))
 */
export type QueriesOptionsOrGetter<
  TQueries extends Array<QueryOptions<any, any, any, any>> = Array<
    QueryOptions<any, any, any, any>
  >,
  TCombinedResult = Array<QueryObserverResult<any, any>>,
> =
  | QueriesOptions<TQueries, TCombinedResult>
  | (() => QueriesOptions<TQueries, TCombinedResult>)

// ═══════════════════════════════════════════════════════════════════════════════
// MUTATION OPTIONS
// Matches React Query's UseMutationOptions
// ═══════════════════════════════════════════════════════════════════════════════

/**
 * Mutation options.
 * This is the MVVM equivalent of React Query's UseMutationOptions.
 */
export interface MutationOptions<
  TData = unknown,
  TError = DefaultError,
  TVariables = void,
  TContext = unknown,
> extends OmitKeyof<
    MutationObserverOptions<TData, TError, TVariables, TContext>,
    '_defaulted'
  > {
  /**
   * Called on EVERY mutation state change.
   * 
   * @example
   * onStateChange: (result) => {
   *   this.updateState({
   *     isSaving: result.isPending,
   *     saveError: result.error?.message ?? null,
   *   })
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

// ═══════════════════════════════════════════════════════════════════════════════
// mutationOptions() HELPER
// Matches React Query's mutationOptions
// ═══════════════════════════════════════════════════════════════════════════════

/**
 * Type-safe mutation options factory.
 * 
 * @example
 * const updateUserMutation = mutationOptions({
 *   mutationKey: ['updateUser'],
 *   mutationFn: (data: UpdateUserInput) => updateUser(data),
 * })
 * 
 * // Use in ViewModel
 * private updateMutation = this.mutation({
 *   ...updateUserMutation,
 *   onSuccess: () => this.invalidateQueries({ queryKey: ['user'] }),
 *   onStateChange: (result) => this.updateState({ isSaving: result.isPending })
 * })
 */

// Overload 1: With mutationKey
export function mutationOptions<
  TData = unknown,
  TError = DefaultError,
  TVariables = void,
  TContext = unknown,
>(
  options: WithRequired<
    MutationOptions<TData, TError, TVariables, TContext>,
    'mutationKey'
  >,
): WithRequired<MutationOptions<TData, TError, TVariables, TContext>, 'mutationKey'>

// Overload 2: Without mutationKey
export function mutationOptions<
  TData = unknown,
  TError = DefaultError,
  TVariables = void,
  TContext = unknown,
>(
  options: Omit<MutationOptions<TData, TError, TVariables, TContext>, 'mutationKey'>,
): Omit<MutationOptions<TData, TError, TVariables, TContext>, 'mutationKey'>

// Overload 3: General case
export function mutationOptions<
  TData = unknown,
  TError = DefaultError,
  TVariables = void,
  TContext = unknown,
>(
  options: MutationOptions<TData, TError, TVariables, TContext>,
): MutationOptions<TData, TError, TVariables, TContext>

// Implementation
export function mutationOptions(options: unknown) {
  return options
}

// ═══════════════════════════════════════════════════════════════════════════════
// RE-EXPORTS from @tanstack/query-core
// ═══════════════════════════════════════════════════════════════════════════════

export type {
  QueryKey,
  QueryObserverResult,
  MutationObserverResult,
  DefaultError,
  DataTag,
  SkipToken,
} from '@tanstack/query-core'
```

---

## 3. queryClient.ts - Singleton Management

```typescript
// queryClient.ts
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

---

## 4. QueryCapability.ts - Core Adapter

```typescript
// QueryCapability.ts
import {
  QueryObserver,
  QueriesObserver,
  MutationObserver,
  notifyManager,
  type QueryKey,
  type QueryObserverOptions,
  type QueryObserverResult,
  type MutationObserverResult,
  type QueriesObserverOptions,
} from '@tanstack/query-core'
import { getQueryClient } from './queryClient'
import type {
  QueryOptions,
  QueryOptionsOrGetter,
  QueriesOptions,
  QueriesOptionsOrGetter,
  MutationOptions,
  MutationHandle,
} from './types'

// ═══════════════════════════════════════════════════════════════════════════════
// Internal Storage Types
// ═══════════════════════════════════════════════════════════════════════════════

/**
 * Storage for a single query (this.query)
 */
interface QueryEntry {
  getOptions: () => QueryOptions<any, any, any, any>
  observer: QueryObserver<any, any, any, any, any>
  unsubscribe: () => void
}

/**
 * Storage for multiple queries (this.queryMultiple)
 * 
 * Uses QueriesObserver - a SINGLE observer that manages multiple queries.
 * This matches how React's useQueries works internally.
 */
interface QueriesEntry<TCombinedResult = any> {
  getOptions: () => QueriesOptions<any, TCombinedResult>
  observer: QueriesObserver<TCombinedResult>
  unsubscribe: () => void
}

/**
 * Storage for a mutation
 */
interface MutationEntry {
  observer: MutationObserver<any, any, any, any>
  unsubscribe: () => void
}

// ═══════════════════════════════════════════════════════════════════════════════
// QueryCapability Class
// ═══════════════════════════════════════════════════════════════════════════════

/**
 * QueryCapability - The MVVM adapter for TanStack Query.
 * 
 * Provides two main query methods:
 * 
 * 1. fetch() - Single query (equivalent to useQuery)
 *    - Creates one QueryObserver
 *    - One subscription per query
 * 
 * 2. fetchMultiple() - Multiple queries (equivalent to useQueries)
 *    - Creates one QueriesObserver (NOT multiple QueryObservers!)
 *    - Single subscription for all queries
 *    - Queries run in PARALLEL
 *    - Optional combine() to transform results
 */
export class QueryCapability {
  private queries: QueryEntry[] = []
  private queriesObservers: QueriesEntry[] = []
  private mutations: MutationEntry[] = []
  private isDestroyed = false

  // ═══════════════════════════════════════════════════════════════════════════
  // fetch() - Single Query (equivalent to useQuery)
  // ═══════════════════════════════════════════════════════════════════════════

  /**
   * Fetch a single query.
   * This is the MVVM equivalent of useQuery().
   * 
   * Creates a QueryObserver that subscribes to a single Query in the cache.
   * 
   * @param optionsOrGetter - Static options or function returning options
   * @returns Promise resolving to data (for imperative usage)
   */
  async fetch<
    TQueryFnData = unknown,
    TError = Error,
    TData = TQueryFnData,
    TQueryKey extends QueryKey = QueryKey,
  >(
    optionsOrGetter: QueryOptionsOrGetter<TQueryFnData, TError, TData, TQueryKey>
  ): Promise<TData | undefined> {
    this.ensureNotDestroyed()
    
    const client = getQueryClient()
    
    // Normalize to function
    const getOptions = typeof optionsOrGetter === 'function'
      ? optionsOrGetter
      : () => optionsOrGetter
    
    const options = getOptions()
    
    // Apply ALL defaults (staleTime, retry, refetchOnWindowFocus, etc.)
    // This matches what useBaseQuery does in React Query
    const defaultedOptions = client.defaultQueryOptions(
      options as QueryObserverOptions
    )
    defaultedOptions._optimisticResults = 'optimistic'
    
    // Create QueryObserver for this query
    const observer = new QueryObserver(client, defaultedOptions)
    
    // Get optimistic result (handles placeholderData, initialData, cached data)
    const optimisticResult = observer.getOptimisticResult(defaultedOptions)
    
    // Fire onStateChange immediately with initial state
    if (options.onStateChange) {
      options.onStateChange(optimisticResult as QueryObserverResult<TData, TError>)
    }
    
    // Subscribe for future updates with batching
    const unsubscribe = observer.subscribe(
      notifyManager.batchCalls(() => {
        if (this.isDestroyed) return
        
        const currentOptions = getOptions()
        currentOptions.onStateChange?.(
          observer.getCurrentResult() as QueryObserverResult<TData, TError>
        )
      })
    )
    
    // Handle race condition between observer creation and subscription
    observer.updateResult()
    
    // Store for cleanup and dynamic updates
    this.queries.push({
      getOptions: getOptions as () => QueryOptions<any, any, any, any>,
      observer,
      unsubscribe,
    })
    
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

  // ═══════════════════════════════════════════════════════════════════════════
  // fetchMultiple() - Multiple Queries (equivalent to useQueries)
  // ═══════════════════════════════════════════════════════════════════════════

  /**
   * Fetch multiple queries in parallel with a single observer.
   * This is the MVVM equivalent of useQueries().
   * 
   * KEY DIFFERENCES FROM CALLING fetch() MULTIPLE TIMES:
   * 
   * 1. Uses QueriesObserver (not multiple QueryObservers)
   *    - Single observer manages all queries internally
   *    - Matches React's useQueries implementation exactly
   * 
   * 2. Single subscription for ALL queries
   *    - When ANY query changes, you get ALL results
   *    - Makes it easy to derive combined states
   * 
   * 3. Atomic updates via setQueries()
   *    - When options change, all queries update together
   *    - Handles adding, removing, reordering queries
   * 
   * 4. Optional combine() function
   *    - Transform array of results into custom shape
   * 
   * 5. Queries run in PARALLEL
   *    - All HTTP requests start simultaneously
   *    - Total time = slowest query (not sum of all)
   * 
   * @param optionsOrGetter - Static options or function returning options
   * @returns Promise resolving to combined result
   */
  async fetchMultiple<
    TQueries extends Array<QueryOptions<any, any, any, any>>,
    TCombinedResult = Array<QueryObserverResult<any, any>>
  >(
    optionsOrGetter: QueriesOptionsOrGetter<TQueries, TCombinedResult>
  ): Promise<TCombinedResult> {
    this.ensureNotDestroyed()
    
    const client = getQueryClient()
    
    // Normalize to function (enables dynamic queries)
    const getOptions = typeof optionsOrGetter === 'function'
      ? optionsOrGetter
      : () => optionsOrGetter
    
    const options = getOptions()
    
    // STEP 1: Apply defaults to ALL queries
    // (Matches React: queries.map(opts => client.defaultQueryOptions(opts)))
    const defaultedQueries = options.queries.map((queryOpts) => {
      const defaulted = client.defaultQueryOptions(queryOpts as QueryObserverOptions)
      defaulted._optimisticResults = 'optimistic'
      return defaulted
    })
    
    // STEP 2: Create SINGLE QueriesObserver
    // (Matches React: new QueriesObserver(client, defaultedQueries, { combine }))
    // 
    // IMPORTANT: This is NOT multiple QueryObservers!
    // QueriesObserver internally creates and manages QueryObservers for each query.
    const observer = new QueriesObserver<TCombinedResult>(
      client,
      defaultedQueries,
      { combine: options.combine } as QueriesObserverOptions<TCombinedResult>
    )
    
    // STEP 3: Get optimistic result BEFORE subscribing
    // (Matches React: observer.getOptimisticResult(defaultedQueries, combine))
    const [, getCombinedResult, trackResult] = 
      observer.getOptimisticResult(defaultedQueries, options.combine)
    
    // Fire onStateChange immediately with initial state
    if (options.onStateChange) {
      options.onStateChange(getCombinedResult(trackResult()))
    }
    
    // STEP 4: Subscribe with batching
    // (Matches React: observer.subscribe(notifyManager.batchCalls(onStoreChange)))
    // 
    // This single subscription fires whenever ANY query's state changes.
    // Each invocation receives ALL results.
    const unsubscribe = observer.subscribe(
      notifyManager.batchCalls(() => {
        if (this.isDestroyed) return
        
        const currentOptions = getOptions()
        currentOptions.onStateChange?.(
          observer.getCurrentResult() as TCombinedResult
        )
      })
    )
    
    // Store for cleanup and dynamic updates
    this.queriesObservers.push({
      getOptions: getOptions as () => QueriesOptions<any, TCombinedResult>,
      observer,
      unsubscribe,
    })
    
    return getCombinedResult(trackResult())
  }

  // ═══════════════════════════════════════════════════════════════════════════
  // onStateChange() - Called when ViewModel state$ changes
  // ═══════════════════════════════════════════════════════════════════════════

  /**
   * Called by BaseViewModel when state$ changes.
   * Re-evaluates ALL query options and updates observers.
   * 
   * This is the MVVM equivalent of React component re-rendering.
   */
  onStateChange(): void {
    if (this.isDestroyed) return
    
    const client = getQueryClient()
    
    // Update single queries (from fetch())
    // (Matches React: observer.setOptions(defaultedOptions))
    for (const entry of this.queries) {
      const options = entry.getOptions()
      const defaultedOptions = client.defaultQueryOptions(
        options as QueryObserverOptions
      )
      defaultedOptions._optimisticResults = 'optimistic'
      entry.observer.setOptions(defaultedOptions)
    }
    
    // Update multiple queries (from fetchMultiple())
    // (Matches React: observer.setQueries(defaultedQueries, options))
    // 
    // IMPORTANT: Use setQueries() NOT individual setOptions()!
    // setQueries() handles adding, removing, and reordering queries atomically.
    for (const entry of this.queriesObservers) {
      const options = entry.getOptions()
      
      const defaultedQueries = options.queries.map((queryOpts) => {
        const defaulted = client.defaultQueryOptions(queryOpts as QueryObserverOptions)
        defaulted._optimisticResults = 'optimistic'
        return defaulted
      })
      
      entry.observer.setQueries(
        defaultedQueries,
        { combine: options.combine } as QueriesObserverOptions
      )
    }
  }

  // ═══════════════════════════════════════════════════════════════════════════
  // Mutation Support
  // ═══════════════════════════════════════════════════════════════════════════

  /**
   * Create a mutation.
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

  // ═══════════════════════════════════════════════════════════════════════════
  // Utility Methods
  // ═══════════════════════════════════════════════════════════════════════════

  /**
   * Prefetch data (cache warming, no observer).
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

  // ═══════════════════════════════════════════════════════════════════════════
  // Lifecycle
  // ═══════════════════════════════════════════════════════════════════════════

  /**
   * Clean up all observers and subscriptions.
   * MUST be called in ViewModel.destroy().
   */
  destroy(): void {
    this.isDestroyed = true
    
    for (const entry of this.queries) {
      entry.unsubscribe()
    }
    this.queries = []
    
    for (const entry of this.queriesObservers) {
      entry.unsubscribe()
    }
    this.queriesObservers = []
    
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

---

## 5. BaseViewModel.ts - Integration

```typescript
// BaseViewModel.ts
import { BehaviorSubject, type Subscription } from 'rxjs'
import { skip } from 'rxjs/operators'
import { QueryCapability } from '../query/QueryCapability'
import type {
  QueryKey,
  QueryOptions,
  QueryOptionsOrGetter,
  QueriesOptions,
  QueriesOptionsOrGetter,
  MutationOptions,
  MutationHandle,
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
  // Single Query API
  // ═══════════════════════════════════════════════════════════════════════════
  
  /**
   * Fetch a single query.
   * This is the MVVM equivalent of useQuery().
   */
  protected query<
    TQueryFnData = unknown,
    TError = Error,
    TData = TQueryFnData,
    TQueryKey extends QueryKey = QueryKey,
  >(
    optionsOrGetter: QueryOptionsOrGetter<TQueryFnData, TError, TData, TQueryKey>
  ): Promise<TData | undefined> {
    return this.__queryCapability.fetch(optionsOrGetter)
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // Multiple Queries API
  // ═══════════════════════════════════════════════════════════════════════════
  
  /**
   * Fetch multiple queries in parallel.
   * This is the MVVM equivalent of useQueries().
   * 
   * All queries run in PARALLEL - total time equals slowest query.
   * onStateChange fires MULTIPLE TIMES as each query resolves.
   */
  protected queryMultiple<
    TQueries extends Array<QueryOptions<any, any, any, any>>,
    TCombinedResult = Array<QueryObserverResult<any, any>>
  >(
    optionsOrGetter: QueriesOptionsOrGetter<TQueries, TCombinedResult>
  ): Promise<TCombinedResult> {
    return this.__queryCapability.fetchMultiple(optionsOrGetter)
  }
  
  // ═══════════════════════════════════════════════════════════════════════════
  // Mutation API
  // ═══════════════════════════════════════════════════════════════════════════
  
  /**
   * Create a mutation.
   * This is the MVVM equivalent of useMutation().
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
  
  // ═══════════════════════════════════════════════════════════════════════════
  // Cache Utilities
  // ═══════════════════════════════════════════════════════════════════════════
  
  /** Prefetch data (cache warming, no observer) */
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

---

## 6. Query Factories Pattern

### voiceQueries.ts

```typescript
// queries/voiceQueries.ts
import { queryOptions, mutationOptions } from '../query/types'
import { VoiceAPI } from '../api/VoiceAPI'
import type { Voice, VoiceFilters, CreateVoiceInput, UpdateVoiceInput } from '../types'

/**
 * Voice query factory with hierarchical keys for granular invalidation.
 * 
 * Key hierarchy:
 *   ['voices']                           - all voice queries
 *   ['voices', 'list']                   - all list queries
 *   ['voices', 'list', filters]          - specific filtered list
 *   ['voices', 'detail']                 - all detail queries
 *   ['voices', 'detail', id]             - specific voice detail
 */
export const voiceQueries = {
  // ─────────────────────────────────────────────────────────────────────────
  // Key-only entries (for invalidation)
  // ─────────────────────────────────────────────────────────────────────────
  
  all: () => ['voices'] as const,
  
  lists: () => [...voiceQueries.all(), 'list'] as const,
  
  details: () => [...voiceQueries.all(), 'detail'] as const,
  
  // ─────────────────────────────────────────────────────────────────────────
  // Query options (for fetching)
  // ─────────────────────────────────────────────────────────────────────────
  
  list: (filters?: VoiceFilters) =>
    queryOptions({
      queryKey: [...voiceQueries.lists(), filters] as const,
      queryFn: () => VoiceAPI.getVoices(filters),
      staleTime: 5 * 60 * 1000,  // 5 minutes
    }),
  
  detail: (id: string) =>
    queryOptions({
      queryKey: [...voiceQueries.details(), id] as const,
      queryFn: () => VoiceAPI.getVoice(id),
      staleTime: 10 * 60 * 1000,  // 10 minutes
    }),
  
  // ─────────────────────────────────────────────────────────────────────────
  // Mutation options
  // ─────────────────────────────────────────────────────────────────────────
  
  mutations: {
    create: () => mutationOptions({
      mutationKey: ['voices', 'create'],
      mutationFn: (data: CreateVoiceInput) => VoiceAPI.create(data),
    }),
    
    update: () => mutationOptions({
      mutationKey: ['voices', 'update'],
      mutationFn: (input: { id: string; data: UpdateVoiceInput }) =>
        VoiceAPI.update(input.id, input.data),
    }),
    
    delete: () => mutationOptions({
      mutationKey: ['voices', 'delete'],
      mutationFn: (id: string) => VoiceAPI.delete(id),
    }),
  },
}
```

---

## 7. Usage Examples

### 7.1 Basic Single Query

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
      ...voiceQueries.list(),
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

### 7.2 Dynamic Query (State-Dependent)

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
        ...voiceQueries.list({ page, search }),
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
  }
  
  setSearch(search: string) {
    this.updateState({ search, page: 1 })
  }
}
```

### 7.3 Multiple Parallel Queries

```typescript
interface DashboardState {
  users: User[]
  posts: Post[]
  settings: Settings | null
  isLoading: boolean
}

class DashboardViewModel extends BaseViewModel<DashboardState> {
  async initialize() {
    await this.queryMultiple({
      queries: [
        { queryKey: ['users'], queryFn: fetchUsers },
        { queryKey: ['posts'], queryFn: fetchPosts },
        { queryKey: ['settings'], queryFn: fetchSettings },
      ],
      combine: (results) => ({
        users: results[0].data ?? [],
        posts: results[1].data ?? [],
        settings: results[2].data ?? null,
        isLoading: results.some(r => r.status === 'pending'),
      }),
      onStateChange: (combined) => {
        this.updateState(combined)
      }
    })
  }
}
```

### 7.4 Dynamic Parallel Queries

```typescript
interface ReposState {
  repos: Repo[]
  allIssues: Issue[]
  totalIssueCount: number
  isLoadingIssues: boolean
}

class ReposViewModel extends BaseViewModel<ReposState> {
  async initialize() {
    // First, load repos
    await this.query({
      queryKey: ['repos'],
      queryFn: fetchRepos,
      onStateChange: (result) => {
        this.updateState({ repos: result.data ?? [] })
      }
    })
    
    // Then, load issues for ALL repos in parallel (dynamic number of queries!)
    await this.queryMultiple(() => ({
      queries: this.getStateValue().repos.map(repo => ({
        queryKey: ['repos', repo.name, 'issues'] as const,
        queryFn: () => fetchIssues(repo.name),
      })),
      combine: (results) => ({
        allIssues: results.flatMap(r => r.data ?? []),
        totalIssueCount: results.reduce((sum, r) => sum + (r.data?.length ?? 0), 0),
        isLoadingIssues: results.some(r => r.status === 'pending'),
      }),
      onStateChange: ({ allIssues, totalIssueCount, isLoadingIssues }) => {
        this.updateState({ allIssues, totalIssueCount, isLoadingIssues })
      }
    }))
  }
}
```

### 7.5 Mutation with Optimistic Update

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
      
      onMutate: async (input) => {
        await this.cancelQueries({ queryKey: ['todos'] })
        
        const previousTodos = this.getQueryData<Todo[]>(['todos'])
        
        const optimisticTodo: Todo = {
          id: `temp-${Date.now()}`,
          text: input.text,
          completed: false,
        }
        
        this.setQueryData<Todo[]>(['todos'], (old) => 
          old ? [...old, optimisticTodo] : [optimisticTodo]
        )
        
        return { previousTodos }
      },
      
      onError: (error, input, context) => {
        if (context?.previousTodos) {
          this.setQueryData(['todos'], context.previousTodos)
        }
      },
      
      onSettled: () => {
        this.invalidateQueries({ queryKey: ['todos'] })
      },
      
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

---

## 8. Platform Configuration

### React Native Setup

```typescript
// app/querySetup.ts
import { AppState, type AppStateStatus } from 'react-native'
import NetInfo from '@react-native-community/netinfo'
import { focusManager, onlineManager } from '@tanstack/query-core'
import { initializeQueryClient } from './core/query/queryClient'

export function setupQueryClient() {
  // Configure focus manager for React Native
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

## 9. Testing Strategy

```typescript
import { QueryClient } from '@tanstack/query-core'
import { QueryCapability } from './QueryCapability'

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
      queryKey: ['test'],
      queryFn: () => Promise.resolve('data'),
      onStateChange: onStateChange2,
    })
    
    expect(onStateChange1).toHaveBeenCalled()
    expect(onStateChange2).toHaveBeenCalled()
  })
  
  it('fetchMultiple runs queries in parallel', async () => {
    const startTimes: number[] = []
    
    await capability.fetchMultiple({
      queries: [
        {
          queryKey: ['q1'],
          queryFn: async () => {
            startTimes.push(Date.now())
            await new Promise(r => setTimeout(r, 50))
            return 'q1'
          },
        },
        {
          queryKey: ['q2'],
          queryFn: async () => {
            startTimes.push(Date.now())
            await new Promise(r => setTimeout(r, 50))
            return 'q2'
          },
        },
      ],
      onStateChange: jest.fn(),
    })
    
    // Both should start within ~5ms of each other (parallel)
    expect(Math.abs(startTimes[0] - startTimes[1])).toBeLessThan(10)
  })
})
```

---

## Summary

### Files and Line Counts

| File | Lines | Purpose |
|------|-------|---------|
| types.ts | ~250 | Type definitions matching React Query |
| queryClient.ts | ~60 | Singleton management |
| QueryCapability.ts | ~300 | Core adapter with fetch() and fetchMultiple() |
| BaseViewModel.ts | ~100 | Integration layer |
| **Total** | **~710** | Complete TanStack Query MVVM integration |

### What We Implemented

| Method | React Query Equivalent | Observer Type |
|--------|----------------------|---------------|
| `this.query()` | `useQuery()` | `QueryObserver` |
| `this.queryMultiple()` | `useQueries()` | `QueriesObserver` |
| `this.mutation()` | `useMutation()` | `MutationObserver` |

### Key Implementation Details

| Aspect | Implementation |
|--------|----------------|
| Single queries | One `QueryObserver` per `query()` call |
| Multiple queries | One `QueriesObserver` for all queries |
| Option updates | `setOptions()` for single, `setQueries()` for multiple |
| Parallel execution | ✅ All HTTP requests start simultaneously |
| Combined state | ✅ Single callback with all results |
| Dynamic queries | ✅ Function-based options, re-evaluated on state change |
| Type safety | ✅ `DataTag` on queryKey for cache type inference |

---

*For architecture explanations, see: `tanstack-query-mvvm-architecture.md`*
