# Settings App Data Caching Layer
## Technical Design Document

**Author:** [Your Name]  
**Date:** December 2024  
**Status:** Draft - For Review  
**Version:** 2.0

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Statement](#2-problem-statement)
3. [Use Cases](#3-use-cases)
4. [Requirements](#4-requirements)
5. [Current Architecture](#5-current-architecture)
6. [Deep Dive: Why Direct Persistent Storage Fails](#6-deep-dive-why-direct-persistent-storage-fails)
7. [Deep Dive: What SWR Actually Requires](#7-deep-dive-what-swr-actually-requires)
8. [The External Invalidation Problem](#8-the-external-invalidation-problem)
9. [Proposed Solutions](#9-proposed-solutions)
10. [Solution Comparison](#10-solution-comparison)
11. [Recommended Approach: TanStack Query Core](#11-recommended-approach-tanstack-query-core)
12. [Implementation Plan](#12-implementation-plan)
13. [Risks and Mitigations](#13-risks-and-mitigations)
14. [Appendices](#14-appendices)

---

## 1. Executive Summary

### The Problem
The Settings application makes redundant API calls on every screen navigation, causing unnecessary loading states. Users see "Loading..." for data they just viewed moments ago.

### The Root Cause
No intelligent caching layer exists between ViewModels and APIs. The current approach triggers fresh API calls on screen focus events, regardless of whether valid cached data exists.

### The Critical Constraint
ViewModel constructors run **synchronously**, but persistent storage (AsyncStorage) is **asynchronous** (~60ms read time). This fundamental mismatch means:
- Direct persistent storage reads cannot populate initial BehaviorSubject state
- Users will always see a loading flash on screen mount
- No amount of architectural cleverness around AsyncStorage can solve this

### The Recommendation
Implement **TanStack Query Core** with a custom RxJS adapter. This provides:
- Synchronous in-memory cache access (no loading flash)
- Battle-tested SWR, deduplication, and observer patterns
- Optional persistence layer for cold start optimization
- ~13KB bundle size, maintained by a large open-source community

---

## 2. Problem Statement

### 2.1 Overview

The Settings application currently makes redundant API calls every time a user navigates between screens, causing unnecessary loading states even when valid data was recently fetched.

### 2.2 Customer-Visible Problems

| Problem | Customer Experience |
|---------|---------------------|
| **Redundant Loading States** | User sees "Loading..." even when returning to a screen they just left |
| **Wasted Time** | User waits for API calls that return the same data |
| **Inconsistent Feel** | App feels slow and unresponsive compared to native apps |
| **Battery/Data Drain** | Unnecessary network calls consume resources |

### 2.3 Root Cause Analysis

```
┌─────────────────────────────────────────────────────────────────┐
│  WHY DOES THIS HAPPEN?                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Current Flow:                                                   │
│    1. User navigates to Screen A                                 │
│    2. ViewModel A fetches data, shows loading, then data         │
│    3. User navigates to Screen B (Screen A may stay mounted)     │
│    4. User navigates back to Screen A                            │
│    5. Focus listener triggers API refetch                        │
│    6. User sees "Loading..." for data they JUST SAW              │
│                                                                  │
│  Missing:                                                        │
│    - Cache to store fetched data                                 │
│    - Staleness logic to determine if data is still valid         │
│    - Observer pattern to sync data across screens                │
│    - Request deduplication to prevent duplicate API calls        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.4 The Synchronous vs Asynchronous Challenge

This is the **fundamental technical constraint** that eliminates certain solution approaches:

```
┌─────────────────────────────────────────────────────────────────┐
│  THE CORE CONSTRAINT                                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ViewModel constructor:                                          │
│    constructor() {                                               │
│      // SYNCHRONOUS - runs immediately, blocks until complete    │
│      this.state = new BehaviorSubject(initialState);             │
│                                                                  │
│      // What is initialState?                                    │
│      // We MUST set it NOW, before any async operation           │
│    }                                                             │
│                                                                  │
│  Problem with AsyncStorage (60ms read):                          │
│    constructor() {                                               │
│      // Cannot await in constructor!                             │
│      this.state = new BehaviorSubject({loading: true}); // STUCK │
│                                                                  │
│      // Even if we call async read...                            │
│      this.loadFromStorage(); // Returns Promise, not data        │
│    }                                                             │
│                                                                  │
│  Result: First frame ALWAYS shows loading state                  │
│                                                                  │
│  Solution requires SYNCHRONOUS cache access:                     │
│    constructor() {                                               │
│      const cached = memoryCache.get('key'); // SYNC! ~0.001ms    │
│      this.state = new BehaviorSubject({                          │
│        data: cached,                                             │
│        loading: !cached                                          │
│      });                                                         │
│    }                                                             │
│                                                                  │
│  Result: First frame shows cached data if available              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Use Cases

### 3.1 Use Case 1: Alexa Voice Selection

**Flow Description:**

```
User Journey:
┌──────────────────────────────────────────────────────────────┐
│ 1. User on Alexa Page, sees "Alexa Voice: Default"           │
│ 2. Taps "Alexa Voice" row                                    │
│ 3. Voice Selection Screen opens (Alexa Page stays mounted)   │
│ 4. User selects "Voice 3"                                    │
│ 5. API call saves selection                                  │
│ 6. User presses BACK                                         │
│ 7. Returns to Alexa Page                                     │
│                                                              │
│ CURRENT BEHAVIOR:                                            │
│ - Focus listener triggers API refetch                        │
│ - User sees "Loading..."                                     │
│ - Data arrives, shows "Voice 3"                              │
│                                                              │
│ DESIRED BEHAVIOR:                                            │
│ - Cache already updated by Voice Selection Screen            │
│ - User sees "Voice 3" instantly                              │
│ - Background refresh happens if data is stale                │
└──────────────────────────────────────────────────────────────┘
```

**Key Technical Points:**
- Outer screen (Alexa Page) remains mounted when inner screen opens
- Both screens need access to the same voice data
- When inner screen mutates data, outer screen must reflect change immediately
- Currently using focus listener to detect change (causes loading state)

### 3.2 Use Case 2: Device Location/Address Edit

**Flow Description:**

```
User Journey:
┌──────────────────────────────────────────────────────────────┐
│ 1. User on Device Options                                    │
│ 2. Taps "About This Device"                                  │
│ 3. Sees Address: "123 Main St"                               │
│ 4. Taps to edit address                                      │
│ 5. Edit form opens                                           │
│ 6. User changes to "456 New St"                              │
│ 7. Taps SAVE → API responds (possibly with address dialog)   │
│ 8. User confirms, navigates back                             │
│                                                              │
│ CURRENT: Focus listener refetches → Loading → New address    │
│ DESIRED: Cache updated after save → New address instantly    │
└──────────────────────────────────────────────────────────────┘
```

### 3.3 Use Case 3: Cold Start (App Previously Killed)

```
User Journey:
┌──────────────────────────────────────────────────────────────┐
│ 1. User had app open previously, viewed Device Settings      │
│ 2. App was killed (user closed, system killed for memory)    │
│ 3. User reopens app                                          │
│ 4. Navigates to Device Settings                              │
│                                                              │
│ WITHOUT PERSISTENCE:                                         │
│ - In-memory cache is empty                                   │
│ - Must fetch from API → Loading state                        │
│                                                              │
│ WITH PERSISTENCE:                                            │
│ - Cache restored from storage during app init                │
│ - User sees cached data instantly                            │
│ - Background refresh validates/updates data                  │
└──────────────────────────────────────────────────────────────┘
```

### 3.4 Use Case 4: External Invalidation (Critical Edge Case)

```
Scenario:
┌──────────────────────────────────────────────────────────────┐
│ 1. User views Device Settings (data cached and persisted)   │
│ 2. App is killed                                             │
│ 3. EXTERNAL EVENT: Device is factory reset / deregistered    │
│ 4. User reopens app                                          │
│                                                              │
│ CRITICAL REQUIREMENT:                                        │
│ - Stale cached data CANNOT be shown, even for 1 frame        │
│ - Showing "Device: Living Room Echo" when device is gone     │
│   would confuse users and cause support calls                │
│                                                              │
│ SOLUTION NEEDED:                                             │
│ - Background task detects external event                     │
│ - Invalidates/clears persistent cache BEFORE app launches    │
│ - OR: Startup validation gate blocks UI until verified       │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. Requirements

### 4.1 Priority 0 (Must Have)

| ID | Requirement | Rationale |
|----|-------------|-----------|
| P0-1 | **Synchronous Cache Access** | Cache reads must be instant to populate ViewModel initial state without loading flash |
| P0-2 | **Cross-Screen Synchronization** | When one screen updates data, all mounted screens showing same data must update automatically |
| P0-3 | **Stale-While-Revalidate (SWR)** | Show cached data immediately, fetch fresh data in background if stale, update UI seamlessly |
| P0-4 | **Time-Based Staleness (TTL)** | Configurable threshold per query type to determine when cached data is "stale" |
| P0-5 | **Manual Invalidation** | Ability to programmatically invalidate specific cache entries after mutations |
| P0-6 | **RxJS Compatibility** | Must integrate with existing BehaviorSubject/Observable patterns in ViewModels |
| P0-7 | **Mutation Support** | Cache updates after successful mutations, with automatic observer notification |

### 4.2 Priority 1 (Should Have)

| ID | Requirement | Rationale |
|----|-------------|-----------|
| P1-1 | **Request Deduplication** | Multiple simultaneous requests for same data should share single API call |
| P1-2 | **Garbage Collection** | Automatic cleanup of unused cache entries after configurable time |
| P1-3 | **Hierarchical Invalidation** | Invalidate groups of related queries (e.g., all "device/*" queries) |
| P1-4 | **Persistent Cache (Cold Start)** | Optionally restore cache from storage on app restart |
| P1-5 | **Background Refetch Coordination** | Prevent duplicate background refetches across screens |
| P1-6 | **External Invalidation Support** | Background task can invalidate cache before foreground app reads it |

### 4.3 Priority 2 (Nice to Have)

| ID | Requirement | Rationale |
|----|-------------|-----------|
| P2-1 | **Optimistic Updates** | Update UI before mutation completes, rollback on failure |
| P2-2 | **Configurable Retry Logic** | Automatic retry for failed requests with exponential backoff |
| P2-3 | **Offline Mutation Queue** | Queue mutations when offline, execute when online |

---

## 5. Current Architecture

### 5.1 Package Structure

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                         │
│                       (App Package)                          │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                     Feature Layer                            │
│                   (UI Package: Screens, Views)               │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                  Business Logic Layer                        │
│              (Core Package: ViewModels, Services)            │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                  Infrastructure Layer                        │
│              (Platform Package: APIs, Storage)               │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Current ViewModel Pattern

```typescript
class DeviceSettingsViewModel {
  private state = new BehaviorSubject<State>({ loading: true, data: null });
  
  constructor(private api: DeviceApi) {
    this.initialize();
  }
  
  private async initialize() {
    // Problem: This is async, constructor already returned
    // UI already rendered with loading: true
    const data = await this.api.getDeviceSettings();
    this.state.next({ loading: false, data });
  }
  
  onFocus() {
    // Called every time screen gains focus
    // Triggers FULL refetch even if data is 2 seconds old
    this.state.next({ loading: true, data: null });
    this.initialize();
  }
}
```

### 5.3 Current Data Flow Problems

```
Timeline of Current Behavior:
─────────────────────────────────────────────────────────────────────────

T=0ms    User navigates to Device Settings
         ├─ ViewModel constructor runs (SYNC)
         ├─ BehaviorSubject created with {loading: true}
         └─ UI renders "Loading..."

T=1ms    initialize() called (ASYNC)
         └─ API request starts

T=200ms  API responds
         ├─ BehaviorSubject.next({data, loading: false})
         └─ UI renders device settings

T=5000ms User navigates to sub-screen

T=10000ms User presses BACK

T=10001ms onFocus() called
          ├─ BehaviorSubject.next({loading: true}) ← PROBLEM!
          └─ API request starts AGAIN

T=10002ms UI renders "Loading..." ← USER SEES THIS!

T=10200ms API responds (same data)
          └─ UI renders device settings (again)

─────────────────────────────────────────────────────────────────────────

The Problem: Data fetched 10 seconds ago is perfectly valid.
User should see it instantly, not a loading state.
```

---

## 6. Deep Dive: Why Direct Persistent Storage Fails

This section provides an exhaustive analysis of why simply reading from persistent storage (AsyncStorage, SQLite, etc.) is fundamentally insufficient for this use case.

### 6.1 The Latency Problem (Surface Issue)

```
Persistent Storage Read Latency:
─────────────────────────────────────────────────────────────────────────

AsyncStorage (React Native):     ~20-80ms per read
SQLite (React Native):           ~5-30ms per read  
SharedPreferences (Android):     ~1-10ms per read
UserDefaults (iOS):              ~1-5ms per read

In-Memory Map:                   ~0.001ms per read

─────────────────────────────────────────────────────────────────────────

Even the "fastest" persistent storage (UserDefaults at 1ms) is
1000x slower than in-memory access.

At 60fps, each frame is 16.67ms.
A 60ms AsyncStorage read = 3-4 frames of loading state visible.
```

But latency is just the **surface issue**. Even if storage were instant, there are deeper architectural problems.

### 6.2 Architectural Problem #1: No Observer Pattern

Persistent storage is **passive**. It stores bytes and returns them on request. It has no concept of "notify me when this changes."

```
The Cross-Screen Sync Problem:
─────────────────────────────────────────────────────────────────────────

Screen A (mounted)          Storage              Screen B (mounted)
     │                         │                       │
     │  read('address')        │                       │
     │ ───────────────────────>│                       │
     │ <───────────────────────│                       │
     │  "123 Main St"          │                       │
     │                         │                       │
     │                         │    write('address',   │
     │                         │    "456 New St")      │
     │                         │ <─────────────────────│
     │                         │                       │
     │  ??? HOW DOES A KNOW?   │                       │
     │                         │                       │
     │  Storage doesn't notify │                       │
     │  Screen A still shows   │                       │
     │  "123 Main St"          │                       │

─────────────────────────────────────────────────────────────────────────

To solve this with storage alone, you would need to BUILD:
1. A pub/sub system on top of storage
2. Subscription management (who is listening to what keys)
3. Notification batching (don't spam subscribers)
4. Cleanup logic (remove dead subscribers)

This is literally building the observer pattern from scratch.
```

### 6.3 Architectural Problem #2: No Request Deduplication

When two screens mount simultaneously and both need the same data:

```
Without Deduplication:
─────────────────────────────────────────────────────────────────────────

Screen A mounts      Screen B mounts
     │                    │
     │  read('device')    │  read('device')
     │  → cache miss      │  → cache miss
     │                    │
     │  fetch('/api')     │  fetch('/api')   ← TWO API CALLS!
     │ ──────────────────────────────────────────────────────>
     │                    │
     │  response 1        │  response 2
     │ <──────────────────────────────────────────────────────
     │                    │
     │  write('device')   │  write('device')
     │  → potential race  │  → which write wins?

─────────────────────────────────────────────────────────────────────────

Storage has no concept of "there's already a fetch in progress."
You would need to BUILD:
1. In-flight request tracking (Map of pending promises)
2. Request key normalization (same parameters = same request)
3. Promise sharing (return same promise to all callers)
4. Cleanup on completion
```

### 6.4 Architectural Problem #3: No State Machine

Storage stores data. It doesn't track the **state** of data:

```
States That Matter (Storage Cannot Track):
─────────────────────────────────────────────────────────────────────────

Query State Machine:
                                    ┌───────────┐
                                    │  STALE    │
                                    │ (needs    │
                      ┌─────────────│  refresh) │◄────────────┐
                      │             └───────────┘             │
                      │                   │                   │
                      │                   │ trigger fetch     │
                      ▼                   ▼                   │
┌───────────┐   ┌───────────┐      ┌───────────┐       ┌───────────┐
│  INITIAL  │──►│  PENDING  │─────►│ FETCHING  │──────►│  SUCCESS  │
│ (no data) │   │ (loading) │      │ (in-flight│       │  (fresh)  │
└───────────┘   └───────────┘      └───────────┘       └───────────┘
                      │                   │                   │
                      │                   │                   │
                      ▼                   ▼                   │
                ┌───────────┐      ┌───────────┐             │
                │   ERROR   │◄─────│  PAUSED   │             │
                │ (failed)  │      │ (offline) │             │
                └───────────┘      └───────────┘             │
                      │                                      │
                      └──────────────────────────────────────┘
                                    retry

─────────────────────────────────────────────────────────────────────────

Storage only knows: "here's some bytes, or there's nothing"
It cannot answer:
- Is this data currently being fetched?
- Is this data stale?
- Did the last fetch fail? Should we retry?
- Is the network offline?
```

### 6.5 Architectural Problem #4: No Staleness Tracking

Storage stores data, but not **when** it was fetched:

```
What Storage Stores:        What You Actually Need:
─────────────────────────────────────────────────────────────────────────

{                           {
  "device": {                 "device": {
    "name": "Echo",             "data": {
    "room": "Kitchen"             "name": "Echo",
  }                               "room": "Kitchen"
}                               },
                                "dataUpdatedAt": 1701532800000,
                                "status": "success",
                                "fetchStatus": "idle",
                                "isStale": false,
                                "staleTime": 300000,
                                "error": null,
                                "errorUpdatedAt": null,
                                "failureCount": 0,
                                "failureReason": null
                              }
                            }

─────────────────────────────────────────────────────────────────────────

You could store metadata alongside data, but then YOU must:
1. Design the metadata schema
2. Update metadata on every operation
3. Calculate staleness (Date.now() - dataUpdatedAt > staleTime)
4. Handle clock skew and timezone issues
5. Migrate schema when requirements change
```

### 6.6 Architectural Problem #5: No Garbage Collection

Storage accumulates data forever:

```
Storage Over Time:
─────────────────────────────────────────────────────────────────────────

Day 1:  User views Device A
        Storage: { deviceA: {...} }                        Size: 1KB

Day 30: User has viewed 50 devices
        Storage: { deviceA, deviceB, ..., deviceAX }       Size: 50KB

Day 90: User has viewed 200 devices, most never again
        Storage: { ...200 devices... }                     Size: 200KB

─────────────────────────────────────────────────────────────────────────

Storage has no concept of:
- "This entry hasn't been accessed in 30 days"
- "This entry has no active observers"
- "We're running low on storage, evict least recently used"

You would need to BUILD:
1. Access time tracking (update timestamp on every read)
2. Observer counting (who is currently using this data)
3. Eviction policy (LRU, LFU, TTL-based)
4. Scheduled cleanup (when to run GC)
5. Size monitoring (how much storage used)
```

### 6.7 Architectural Problem #6: Race Conditions

Concurrent access to storage creates data consistency issues:

```
Race Condition Scenario:
─────────────────────────────────────────────────────────────────────────

Time    Thread A                    Storage              Thread B
─────────────────────────────────────────────────────────────────────────
T1      read('counter')                                  
                              ──────>
T2                            value = 5
              <──────────────
T3                                                       read('counter')
                                                   ──────>
T4                                                       value = 5
                                                   <──────
T5      value++ (now 6)
        write('counter', 6)
                              ──────>
T6                                                       value++ (now 6)
                                                         write('counter', 6)
                                                   ──────>
T7                            Final value = 6
                              (Should be 7!)

─────────────────────────────────────────────────────────────────────────

This is the classic read-modify-write race condition.
Storage provides no transactions or atomic operations.

You would need to BUILD:
1. Locking mechanism (mutex/semaphore)
2. Transactional writes (all-or-nothing)
3. Optimistic concurrency (version numbers)
4. Conflict resolution (last-write-wins vs merge)
```

### 6.8 Architectural Problem #7: Partial Write Corruption

What happens if the app crashes mid-write?

```
Crash Scenario:
─────────────────────────────────────────────────────────────────────────

T1    Start writing large object to storage
      storage.set('data', JSON.stringify(largeObject))

T2    Write is 50% complete (bytes written to disk)
      
T3    APP CRASH / KILL / BATTERY DIES

T4    App restarts
      
T5    storage.get('data') returns...???
      - Truncated JSON string?
      - Null?
      - Previous value?
      - Corrupted bytes?

─────────────────────────────────────────────────────────────────────────

Different storage implementations handle this differently:
- AsyncStorage: May return truncated data
- SQLite: Transactions rollback, but raw writes don't
- SharedPreferences: Usually atomic, but no guarantees

You would need to BUILD:
1. Write-ahead logging (WAL)
2. Atomic swap (write to temp, rename)
3. Checksums (detect corruption)
4. Recovery logic (fallback to previous value)
```

### 6.9 Architectural Problem #8: Serialization Overhead

Every storage operation requires serialization:

```
Read Operation Cost:
─────────────────────────────────────────────────────────────────────────

In-Memory Cache:
  map.get('key')
  → Direct object reference
  → ~0.001ms
  → Zero CPU

Persistent Storage:
  storage.get('key')
  → Read bytes from disk (~20ms)
  → JSON.parse(bytes) (~5-50ms for large objects)
  → Create new object in memory
  → ~25-70ms total
  → Significant CPU for large objects

─────────────────────────────────────────────────────────────────────────

For a 50KB settings object:
- JSON.parse: ~10ms
- Object creation: ~2ms
- Memory allocation: ~2ms

This happens on EVERY read. No caching of parsed objects.
```

### 6.10 Summary: What You'd Need to Build on Top of Storage

| Feature | Complexity | Estimated Effort |
|---------|------------|------------------|
| Pub/Sub Observer System | High | 3-4 days |
| Request Deduplication | Medium | 2-3 days |
| State Machine | High | 3-4 days |
| Staleness Tracking | Medium | 2-3 days |
| Garbage Collection | Medium | 2-3 days |
| Race Condition Handling | High | 2-3 days |
| Crash Recovery | High | 2-3 days |
| Serialization Optimization | Medium | 1-2 days |
| **Total** | | **~3-4 weeks** |

**And you still have the fundamental latency problem.**

Storage reads are inherently async, which means you cannot populate initial ViewModel state synchronously. **No amount of features built on top of storage can solve this.**

---

## 7. Deep Dive: What SWR Actually Requires

"Stale-While-Revalidate" sounds simple, but implementing it correctly is surprisingly complex.

### 7.1 SWR Conceptual Overview

```
SWR Flow:
─────────────────────────────────────────────────────────────────────────

User requests data for ['device', 'abc123']

┌─────────────────────────────────────────────────────────────────────┐
│ Step 1: Check Cache                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   cache.get(['device', 'abc123'])                                   │
│                                                                      │
│   ┌─── Cache HIT ───┐          ┌─── Cache MISS ───┐                │
│   │                 │          │                  │                 │
│   │  Data exists    │          │  No data         │                 │
│   │  Check staleness│          │  Must fetch      │                 │
│   │                 │          │                  │                 │
│   └────────┬────────┘          └────────┬─────────┘                │
│            │                            │                           │
│            ▼                            ▼                           │
│   ┌─── FRESH ───┐  ┌─── STALE ───┐    ┌─── PENDING ───┐           │
│   │             │  │             │    │               │            │
│   │ Return data │  │ Return data │    │ Show loading  │            │
│   │ No fetch    │  │ AND fetch   │    │ Fetch data    │            │
│   │             │  │ in background│    │               │            │
│   └─────────────┘  └─────────────┘    └───────────────┘            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 Component #1: Timestamp Tracking

Every cached entry needs metadata:

```typescript
interface CacheEntry<T> {
  data: T;
  dataUpdatedAt: number;      // When was this fetched?
  status: 'pending' | 'success' | 'error';
  fetchStatus: 'idle' | 'fetching' | 'paused';
  error: Error | null;
  errorUpdatedAt: number | null;
  staleTime: number;          // How long until stale?
  gcTime: number;             // How long until garbage collected?
  fetchFailureCount: number;
  lastFetchTime: number | null;
}
```

### 7.3 Component #2: Staleness Calculation

```typescript
function isStale(entry: CacheEntry): boolean {
  if (entry.status !== 'success') return true;
  if (entry.isInvalidated) return true;
  
  const now = Date.now();
  const age = now - entry.dataUpdatedAt;
  
  return age > entry.staleTime;
}

// Edge cases to handle:
// - Clock skew (system time changes)
// - Timezone changes
// - Device sleep (time jumps)
// - staleTime of 0 (always stale)
// - staleTime of Infinity (never stale)
```

### 7.4 Component #3: Background Fetch Orchestration

```typescript
async function fetchWithSWR<T>(
  key: QueryKey,
  fetchFn: () => Promise<T>,
  options: QueryOptions
): Promise<T> {
  const cached = cache.get(key);
  
  if (cached && cached.status === 'success') {
    // Return cached data immediately
    notifyObservers(key, cached);
    
    // Check if stale
    if (isStale(cached)) {
      // Background refetch (non-blocking)
      backgroundFetch(key, fetchFn).catch(handleBackgroundError);
    }
    
    return cached.data;
  }
  
  // No cache, must fetch
  return await foregroundFetch(key, fetchFn);
}

// Challenges:
// - How to notify observers when background fetch completes?
// - What if background fetch fails?
// - What if user navigates away mid-fetch?
// - How to cancel orphaned fetches?
```

### 7.5 Component #4: Request Deduplication

```typescript
const inFlightRequests = new Map<string, Promise<any>>();

async function deduplicatedFetch<T>(
  key: QueryKey,
  fetchFn: () => Promise<T>
): Promise<T> {
  const keyHash = hashQueryKey(key);
  
  // Check if request already in flight
  const existing = inFlightRequests.get(keyHash);
  if (existing) {
    return existing; // Share the promise
  }
  
  // Create new request
  const promise = fetchFn()
    .finally(() => {
      inFlightRequests.delete(keyHash);
    });
  
  inFlightRequests.set(keyHash, promise);
  return promise;
}

// Challenges:
// - Query key normalization ({a:1, b:2} === {b:2, a:1}?)
// - What if options differ? (different staleTime)
// - Memory leaks (cleanup on unmount)
```

### 7.6 Component #5: Observer Notification System

```typescript
class QueryObserverManager {
  private observers = new Map<string, Set<Observer>>();
  
  subscribe(key: QueryKey, observer: Observer): () => void {
    const keyHash = hashQueryKey(key);
    
    if (!this.observers.has(keyHash)) {
      this.observers.set(keyHash, new Set());
    }
    
    this.observers.get(keyHash)!.add(observer);
    
    // Return unsubscribe function
    return () => {
      this.observers.get(keyHash)?.delete(observer);
    };
  }
  
  notify(key: QueryKey, result: QueryResult): void {
    const keyHash = hashQueryKey(key);
    const observers = this.observers.get(keyHash);
    
    if (observers) {
      // Batch notifications to avoid render storms
      batchedNotify(() => {
        observers.forEach(observer => observer.onUpdate(result));
      });
    }
  }
}

// Challenges:
// - Notification batching (React concurrent mode)
// - Observer cleanup (memory leaks)
// - Ordering guarantees (which observer first?)
// - Error boundaries (one observer throws, others still work)
```

### 7.7 Component #6: Error Handling

```typescript
async function fetchWithErrorHandling<T>(
  key: QueryKey,
  fetchFn: () => Promise<T>,
  options: QueryOptions
): Promise<T> {
  const entry = cache.get(key);
  
  try {
    const data = await fetchFn();
    
    cache.set(key, {
      data,
      status: 'success',
      error: null,
      fetchFailureCount: 0,
      // ...
    });
    
    return data;
    
  } catch (error) {
    const failureCount = (entry?.fetchFailureCount ?? 0) + 1;
    
    cache.set(key, {
      ...entry,
      status: 'error',
      error,
      fetchFailureCount: failureCount,
      errorUpdatedAt: Date.now(),
    });
    
    // Should we retry?
    if (failureCount < options.retry) {
      const delay = calculateBackoff(failureCount);
      await sleep(delay);
      return fetchWithErrorHandling(key, fetchFn, options);
    }
    
    // Keep stale data if available?
    if (entry?.data && options.keepPreviousData) {
      return entry.data; // Return stale data on error
    }
    
    throw error;
  }
}

// Challenges:
// - Exponential backoff calculation
// - Maximum retry limit
// - Network vs server errors (retry network, not 404)
// - Retry cancellation on unmount
// - Error deduplication (same error, multiple observers)
```

### 7.8 Component #7: Race Condition Prevention

```typescript
// The problem: Response B arrives after Response A,
// but Response A was for a newer request

class QueryFetchManager {
  private fetchCounts = new Map<string, number>();
  
  async fetch<T>(key: QueryKey, fetchFn: () => Promise<T>): Promise<T> {
    const keyHash = hashQueryKey(key);
    
    // Increment fetch count
    const currentCount = (this.fetchCounts.get(keyHash) ?? 0) + 1;
    this.fetchCounts.set(keyHash, currentCount);
    
    const data = await fetchFn();
    
    // Check if this is still the latest fetch
    if (this.fetchCounts.get(keyHash) !== currentCount) {
      // A newer fetch was started, ignore this result
      throw new CancelledError('Superseded by newer fetch');
    }
    
    return data;
  }
}

// Challenges:
// - Fetch count overflow (use incrementing ID instead?)
// - Memory cleanup
// - Cancellation propagation to network layer
```

### 7.9 Summary: SWR Implementation Effort

| Component | Complexity | Edge Cases | Effort |
|-----------|------------|------------|--------|
| Timestamp Tracking | Low | Clock skew, device sleep | 0.5 days |
| Staleness Calculation | Medium | Infinity, 0, invalidation | 1 day |
| Background Fetch | High | Cancellation, errors, ordering | 2-3 days |
| Request Deduplication | Medium | Key normalization, cleanup | 1-2 days |
| Observer System | High | Memory leaks, batching | 2-3 days |
| Error Handling | High | Retry logic, backoff, fallbacks | 2-3 days |
| Race Condition Prevention | Medium | Cancellation, ordering | 1-2 days |
| **Total** | | | **10-15 days** |

And this doesn't include testing, edge cases discovered in production, or maintenance.

---

## 8. The External Invalidation Problem

This is a critical edge case that affects architectural decisions.

### 8.1 The Scenario

```
External Invalidation Flow:
─────────────────────────────────────────────────────────────────────────

Timeline:
├── T1: User views Device Settings (device: "Living Room Echo")
│       Data cached in memory AND persisted to storage
│
├── T2: User puts app in background
│
├── T3: App process killed (by system or user)
│       In-memory cache lost
│       Persistent cache still has device data
│
├── T4: EXTERNAL EVENT OCCURS
│       Examples:
│       - Device factory reset
│       - Device deregistered from account
│       - User logged out from another device
│       - Server-side data migration
│
├── T5: User reopens app
│       
│       QUESTION: What does user see?
│
│       IF we blindly trust persistent cache:
│         → User sees "Living Room Echo" (WRONG!)
│         → Device no longer exists!
│         → Causes confusion, support calls
│
│       WHAT WE NEED:
│         → Detect that cached data is invalid
│         → Show loading or fetch fresh
│         → NEVER show stale data for this case
│
─────────────────────────────────────────────────────────────────────────
```

### 8.2 Why This Is Different From Normal Staleness

Normal SWR says: "Show stale data while fetching fresh."

External invalidation says: "Stale data is **dangerous**, never show it."

```
┌─────────────────────────────────────────────────────────────────────┐
│ NORMAL STALENESS                 │ EXTERNAL INVALIDATION            │
├─────────────────────────────────────────────────────────────────────┤
│                                  │                                   │
│ Data is "outdated"               │ Data is "wrong"                   │
│ Safe to show while refreshing    │ UNSAFE to show at all             │
│                                  │                                   │
│ Example:                         │ Example:                          │
│ - Weather from 5 min ago         │ - Device that no longer exists    │
│ - Stock price from 1 min ago     │ - User who was deleted            │
│ - Settings from last session     │ - Account that was logged out     │
│                                  │                                   │
│ User experience if shown:        │ User experience if shown:         │
│ "Slightly outdated, but fine"    │ "This is completely wrong!"       │
│                                  │                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.3 Solution Approaches

#### Approach A: Background Task Pre-Invalidation

```
Background Task Flow:
─────────────────────────────────────────────────────────────────────────

Background Task runs periodically (or on push notification):

1. Check for invalidation conditions:
   - Call lightweight API: GET /api/user/validation-token
   - Compare with stored token
   - Or: Check device registration status
   - Or: Receive push notification with invalidation command

2. If invalidation needed:
   - Clear specific keys from persistent storage
   - Or: Set invalidation flag in storage
   - Or: Update validation token

3. When foreground app launches:
   - Cache hydration reads from storage
   - Invalidated keys are missing (or flagged)
   - App fetches fresh data instead of showing stale

─────────────────────────────────────────────────────────────────────────

Pros:
- User never sees stale data
- No blocking on app launch

Cons:
- Background task may not run (killed by OS, battery saver)
- Requires background task infrastructure
- Adds complexity
```

#### Approach B: Startup Validation Gate

```
Startup Validation Flow:
─────────────────────────────────────────────────────────────────────────

App Launch Sequence:

1. Show splash screen

2. Read validation token from storage

3. Call validation API:
   GET /api/user/validation-token
   
4. Compare tokens:
   
   ┌─── MATCH ───┐              ┌─── MISMATCH ───┐
   │             │              │                │
   │ Cache is    │              │ Cache is       │
   │ valid       │              │ invalidated    │
   │             │              │                │
   │ Hydrate     │              │ Clear cache    │
   │ cache       │              │ Fetch fresh    │
   │             │              │                │
   └─────────────┘              └────────────────┘

5. Hide splash screen, show app

─────────────────────────────────────────────────────────────────────────

Pros:
- Guaranteed to never show stale data
- Simple to implement

Cons:
- Adds latency to every app launch (~100-500ms)
- Requires network on launch (what if offline?)
- Blocking operation
```

#### Approach C: Versioned Cache Keys

```
Versioned Cache Strategy:
─────────────────────────────────────────────────────────────────────────

Instead of:
  cache key: ['device', 'abc123']
  
Use:
  cache key: ['device', 'abc123', 'v1.2.3']
  
Where version is:
  - App version (invalidate on app update)
  - Data schema version (invalidate on migration)
  - Server-provided token (invalidate on server change)

On launch:
  1. Get current version from server (or use known version)
  2. Cache lookup uses versioned key
  3. Old versions don't match → cache miss → fresh fetch

─────────────────────────────────────────────────────────────────────────

Pros:
- Simple to implement
- No separate invalidation logic

Cons:
- Increases cache size (old versions until GC)
- Requires version in every query key
- Still need to get version from somewhere
```

#### Approach D: Conservative Cache Policy

```
Conservative Strategy:
─────────────────────────────────────────────────────────────────────────

For "critical" data that cannot be stale:
  - Never persist to storage
  - Only use in-memory cache
  - Always fetch fresh on cold start

For "safe" data that can be stale:
  - Persist to storage
  - Use SWR on cold start

─────────────────────────────────────────────────────────────────────────

Implementation:

queryClient.setQueryDefaults(['device', 'critical'], {
  staleTime: 0,         // Always stale
  gcTime: 0,            // Never cache
  meta: { persist: false }
})

queryClient.setQueryDefaults(['settings', 'preferences'], {
  staleTime: 5 * 60 * 1000,  // 5 minutes
  meta: { persist: true }
})

─────────────────────────────────────────────────────────────────────────

Pros:
- Simple mental model
- No complex invalidation logic
- Conservative = safe

Cons:
- No instant display for critical data
- May over-fetch
```

### 8.4 TanStack Query Core in Background Tasks

**Critical Question:** Will TanStack Query Core work in a background task environment with no UI?

**Answer:** Yes.

```
TanStack Query Core Dependencies:
─────────────────────────────────────────────────────────────────────────

@tanstack/query-core uses ONLY:

JavaScript Built-ins:
  - Map, Set, WeakMap
  - Promise
  - setTimeout, clearTimeout
  - Date.now()
  - JSON.stringify (for key hashing)
  - console (optional, for logging)

NO DOM dependencies:
  - No window
  - No document
  - No React
  - No UI framework

─────────────────────────────────────────────────────────────────────────

This means it works in:
  - Node.js
  - React Native background tasks
  - Web Workers
  - Service Workers
  - Any JavaScript runtime
```

**Background Task Usage:**

```typescript
// background-task.ts
import { QueryClient, QueryCache } from '@tanstack/query-core';
import AsyncStorage from '@react-native-async-storage/async-storage';

export async function runBackgroundInvalidation() {
  // Create QueryClient (same as foreground would)
  const queryClient = new QueryClient();
  
  // Read persisted cache from storage
  const persistedCache = await AsyncStorage.getItem('QUERY_CACHE');
  
  if (persistedCache) {
    // Check if invalidation needed
    const shouldInvalidate = await checkInvalidationCondition();
    
    if (shouldInvalidate) {
      // Option 1: Clear entire cache
      await AsyncStorage.removeItem('QUERY_CACHE');
      
      // Option 2: Selective invalidation
      const parsed = JSON.parse(persistedCache);
      const filtered = filterInvalidQueries(parsed);
      await AsyncStorage.setItem('QUERY_CACHE', JSON.stringify(filtered));
    }
  }
}
```

### 8.5 Recommended Approach for External Invalidation

For the Settings app, recommend a **hybrid approach**:

```
Recommended Strategy:
─────────────────────────────────────────────────────────────────────────

1. Classify data into tiers:

   TIER 1: Critical (never show stale)
     - Device registration status
     - Account authentication state
     - User permissions
     
   TIER 2: Important (SWR with short staleTime)
     - Device settings
     - User preferences
     - Recent activity
     
   TIER 3: Static (long cache, persist)
     - Available voices list
     - Region settings
     - Feature flags

2. Implementation:

   TIER 1: 
     - Don't persist
     - staleTime: 0
     - Always fetch on mount
     - Background task not needed (no persistence)
   
   TIER 2:
     - Persist with version key
     - staleTime: 5 minutes
     - Background task updates version if needed
     - On launch, version mismatch → invalidate
   
   TIER 3:
     - Persist indefinitely
     - staleTime: 1 hour
     - Only invalidate on app update

─────────────────────────────────────────────────────────────────────────
```

---

## 9. Proposed Solutions

### 9.1 Option A: Storage-Only with Subscriptions

**Description:** Use the existing async storage layer with its subscription API for cross-screen sync. No in-memory cache.

**Architecture:**

```
┌───────────────────────────────────────────────────────────────────────┐
│                          ViewModels                                    │
│  ┌─────────────────┐              ┌─────────────────┐                 │
│  │  ViewModel A    │              │  ViewModel B    │                 │
│  │  (subscribed)   │              │  (subscribed)   │                 │
│  └────────┬────────┘              └────────┬────────┘                 │
└───────────┼────────────────────────────────┼──────────────────────────┘
            │                                │
            ▼                                ▼
┌───────────────────────────────────────────────────────────────────────┐
│                       Storage Subscription API                         │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  onValueChange('address', callback)                              │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌───────────────────────────────────────────────────────────────────────┐
│                       AsyncStorage (60ms)                              │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  { 'address': '123 Main St', 'device': {...}, ... }             │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
```

**What This Solves:**
| Requirement | Solved? | How |
|-------------|---------|-----|
| P0-2: Cross-Screen Sync | ✅ | Via storage subscriptions |
| P1-4: Persistence | ✅ | Native to storage |

**What This Does NOT Solve:**

| Requirement | Solved? | Why Not |
|-------------|---------|---------|
| P0-1: Sync Cache Access | ❌ | Storage is async (60ms). Cannot populate ViewModel initial state. |
| P0-3: SWR | ❌ | Must build entirely. Storage doesn't track staleness. |
| P0-4: TTL | ❌ | Must build. Storage doesn't expire data. |
| P1-1: Request Dedup | ❌ | Impossible. Storage doesn't know about in-flight requests. |
| P1-2: GC | ❌ | Must build. Storage accumulates forever. |
| P1-5: Refetch Coordination | ❌ | Impossible. No in-flight tracking. |

**The Fundamental Problem:**

```
constructor() {
  // Storage read is async, cannot await in constructor
  const data = storage.get('key'); // Returns Promise, not data!
  
  // Must set initial state NOW
  this.state = new BehaviorSubject({
    loading: true,  // Stuck with loading state
    data: null
  });
}

// RESULT: User ALWAYS sees loading flash on mount
// This cannot be fixed with any amount of clever coding
```

**Effort Estimate:** 2-3 weeks (for SWR, TTL, GC, etc.)

**Verdict:** ❌ Does not meet P0-1 requirement. Eliminates this option.

---

### 9.2 Option B: Custom In-Memory Cache with RxJS

**Description:** Build a custom in-memory caching solution using RxJS primitives (BehaviorSubject, share operators) with optional storage persistence.

**Architecture:**

```
┌───────────────────────────────────────────────────────────────────────┐
│                          ViewModels                                    │
│  ┌─────────────────┐              ┌─────────────────┐                 │
│  │  ViewModel A    │              │  ViewModel B    │                 │
│  └────────┬────────┘              └────────┬────────┘                 │
└───────────┼────────────────────────────────┼──────────────────────────┘
            │                                │
            ▼                                ▼
┌───────────────────────────────────────────────────────────────────────┐
│                    Custom RxJS Cache Layer                             │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  CacheService                                                    │  │
│  │  ├─ cache: Map<string, BehaviorSubject<CacheEntry>>             │  │
│  │  ├─ inFlightRequests: Map<string, Observable>                   │  │
│  │  ├─ get(key): Observable<T>                                     │  │
│  │  ├─ set(key, data): void                                        │  │
│  │  ├─ invalidate(key): void                                       │  │
│  │  └─ getSynchronous(key): T | null                               │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
            │
            ▼ (optional persistence)
┌───────────────────────────────────────────────────────────────────────┐
│                       AsyncStorage                                     │
└───────────────────────────────────────────────────────────────────────┘
```

**Implementation Sketch:**

```typescript
interface CacheEntry<T> {
  data: T | null;
  status: 'pending' | 'success' | 'error';
  error: Error | null;
  dataUpdatedAt: number | null;
  isStale: boolean;
}

class RxJSCacheService {
  private cache = new Map<string, BehaviorSubject<CacheEntry<any>>>();
  private inFlight = new Map<string, Observable<any>>();
  private staleTimeouts = new Map<string, NodeJS.Timeout>();
  
  getSync<T>(key: string): T | null {
    const entry = this.cache.get(key);
    return entry?.getValue().data ?? null;
  }
  
  get<T>(key: string, fetchFn: () => Observable<T>, options: CacheOptions): Observable<CacheEntry<T>> {
    const keyHash = this.hashKey(key);
    
    // Get or create cache entry
    if (!this.cache.has(keyHash)) {
      this.cache.set(keyHash, new BehaviorSubject<CacheEntry<T>>({
        data: null,
        status: 'pending',
        error: null,
        dataUpdatedAt: null,
        isStale: true
      }));
    }
    
    const subject = this.cache.get(keyHash)!;
    const current = subject.getValue();
    
    // If no data or stale, trigger fetch
    if (!current.data || current.isStale) {
      this.triggerFetch(keyHash, fetchFn, options);
    }
    
    return subject.asObservable();
  }
  
  private triggerFetch<T>(keyHash: string, fetchFn: () => Observable<T>, options: CacheOptions): void {
    // Deduplication: check if already fetching
    if (this.inFlight.has(keyHash)) {
      return; // Reuse existing request
    }
    
    const request$ = fetchFn().pipe(
      share(), // Share among subscribers
      finalize(() => this.inFlight.delete(keyHash))
    );
    
    this.inFlight.set(keyHash, request$);
    
    request$.subscribe({
      next: (data) => {
        this.cache.get(keyHash)!.next({
          data,
          status: 'success',
          error: null,
          dataUpdatedAt: Date.now(),
          isStale: false
        });
        
        // Set stale timeout
        this.setStaleTimeout(keyHash, options.staleTime);
      },
      error: (error) => {
        this.cache.get(keyHash)!.next({
          ...this.cache.get(keyHash)!.getValue(),
          status: 'error',
          error
        });
      }
    });
  }
  
  private setStaleTimeout(keyHash: string, staleTime: number): void {
    // Clear existing timeout
    const existing = this.staleTimeouts.get(keyHash);
    if (existing) clearTimeout(existing);
    
    // Set new timeout
    const timeout = setTimeout(() => {
      const subject = this.cache.get(keyHash);
      if (subject) {
        subject.next({
          ...subject.getValue(),
          isStale: true
        });
      }
    }, staleTime);
    
    this.staleTimeouts.set(keyHash, timeout);
  }
  
  invalidate(key: string): void {
    const keyHash = this.hashKey(key);
    const subject = this.cache.get(keyHash);
    if (subject) {
      subject.next({
        ...subject.getValue(),
        isStale: true
      });
    }
  }
  
  // ... GC, persistence, etc.
}
```

**What This Solves:**

| Requirement | Solved? | How |
|-------------|---------|-----|
| P0-1: Sync Cache Access | ✅ | `getSync()` reads from Map |
| P0-2: Cross-Screen Sync | ✅ | BehaviorSubject notifies subscribers |
| P0-6: RxJS Compatibility | ✅ | Native RxJS |

**What Requires Building:**

| Requirement | Complexity | Edge Cases |
|-------------|------------|------------|
| P0-3: SWR | High | Background fetch orchestration, error handling |
| P0-4: TTL | Medium | Timeout management, cleanup |
| P0-5: Invalidation | Low | Key matching, prefix matching |
| P0-7: Mutations | Medium | Optimistic updates, rollback |
| P1-1: Deduplication | Medium | Request normalization |
| P1-2: GC | Medium | Observer counting, timeout tracking |
| P1-3: Hierarchical Invalidation | Medium | Key pattern matching |
| P1-4: Persistence | High | Hydration, dehydration, versioning |

**Effort Estimate:** 3-4 weeks

**Pros:**
- No external dependencies
- Full control over implementation
- Native RxJS integration
- Can tailor to exact needs

**Cons:**
- Significant development effort
- Edge cases will be discovered over time
- Maintenance burden
- Not battle-tested

**Verdict:** ⚠️ Viable, but high effort and risk. Consider if TanStack Query doesn't meet needs.

---

### 9.3 Option C: Zustand with Custom SWR Logic

**Description:** Use Zustand for state management with custom SWR, deduplication, and persistence logic built on top.

**Architecture:**

```
┌───────────────────────────────────────────────────────────────────────┐
│                          ViewModels                                    │
│  ┌─────────────────┐              ┌─────────────────┐                 │
│  │  ViewModel A    │              │  ViewModel B    │                 │
│  └────────┬────────┘              └────────┬────────┘                 │
└───────────┼────────────────────────────────┼──────────────────────────┘
            │                                │
            ▼                                ▼
┌───────────────────────────────────────────────────────────────────────┐
│                    RxJS Adapter Layer                                  │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  Converts Zustand subscriptions → Observables                    │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌───────────────────────────────────────────────────────────────────────┐
│                    Zustand Store                                       │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  queries: {                                                      │  │
│  │    [queryHash]: {                                                │  │
│  │      data, status, error, dataUpdatedAt, staleTime, ...         │  │
│  │    }                                                             │  │
│  │  }                                                               │  │
│  │                                                                  │  │
│  │  inFlightRequests: Map<string, Promise>                          │  │
│  │                                                                  │  │
│  │  fetchQuery(key, fetchFn, options): Promise<T>                   │  │
│  │  invalidateQueries(predicate): void                              │  │
│  │  setQueryData(key, data): void                                   │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
            │
            ▼ (via zustand/middleware/persist)
┌───────────────────────────────────────────────────────────────────────┐
│                       AsyncStorage                                     │
└───────────────────────────────────────────────────────────────────────┘
```

**Why Zustand?**

| Feature | Zustand | Redux |
|---------|---------|-------|
| Bundle size | ~1KB | ~10KB (+ RTK ~15KB) |
| Boilerplate | Minimal | Significant |
| TypeScript | Excellent | Good |
| Middleware | Simple | Complex |
| Learning curve | Low | Medium-High |

**Implementation Sketch:**

```typescript
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

interface QueryState {
  queries: Record<string, QueryEntry>;
  inFlight: Record<string, Promise<any>>;
}

interface QueryEntry {
  data: any;
  status: 'pending' | 'success' | 'error';
  error: Error | null;
  dataUpdatedAt: number | null;
  staleTime: number;
}

const useQueryStore = create<QueryState>()(
  persist(
    (set, get) => ({
      queries: {},
      inFlight: {},
      
      fetchQuery: async (key: string, fetchFn: () => Promise<any>, options: QueryOptions) => {
        const keyHash = hashKey(key);
        const existing = get().queries[keyHash];
        
        // Check if data exists and is fresh
        if (existing?.data && !isStale(existing)) {
          return existing.data;
        }
        
        // Check for in-flight request (deduplication)
        if (get().inFlight[keyHash]) {
          return get().inFlight[keyHash];
        }
        
        // Start new fetch
        const promise = fetchFn();
        set(state => ({
          inFlight: { ...state.inFlight, [keyHash]: promise }
        }));
        
        try {
          const data = await promise;
          set(state => ({
            queries: {
              ...state.queries,
              [keyHash]: {
                data,
                status: 'success',
                error: null,
                dataUpdatedAt: Date.now(),
                staleTime: options.staleTime
              }
            },
            inFlight: omit(state.inFlight, keyHash)
          }));
          return data;
        } catch (error) {
          set(state => ({
            queries: {
              ...state.queries,
              [keyHash]: {
                ...state.queries[keyHash],
                status: 'error',
                error
              }
            },
            inFlight: omit(state.inFlight, keyHash)
          }));
          throw error;
        }
      },
      
      invalidateQueries: (predicate: (key: string) => boolean) => {
        set(state => ({
          queries: Object.fromEntries(
            Object.entries(state.queries).map(([key, entry]) => [
              key,
              predicate(key) ? { ...entry, dataUpdatedAt: 0 } : entry
            ])
          )
        }));
      },
      
      getQuerySync: (key: string) => {
        const keyHash = hashKey(key);
        return get().queries[keyHash]?.data ?? null;
      }
    }),
    {
      name: 'query-cache',
      storage: createJSONStorage(() => AsyncStorage),
      partialize: (state) => ({ queries: state.queries })
    }
  )
);
```

**What This Solves:**

| Requirement | Solved? | How |
|-------------|---------|-----|
| P0-1: Sync Cache Access | ✅ | `getQuerySync()` reads from store |
| P0-2: Cross-Screen Sync | ✅ | Zustand subscriptions |
| P1-4: Persistence | ✅ | Zustand persist middleware |

**What Requires Building:**

Same as Option B, but with Zustand as the foundation instead of raw RxJS.

**Additional Complexity:**
- RxJS adapter layer needed (Zustand → Observable)
- Zustand uses its own subscription model (not RxJS native)

**Effort Estimate:** 3-4 weeks

**Pros:**
- Smaller bundle than Redux
- Zustand persist middleware handles basics
- Growing community and ecosystem

**Cons:**
- Still building SWR, GC, etc. from scratch
- RxJS adapter adds complexity
- Two subscription models to bridge

**Verdict:** ⚠️ Viable, similar trade-offs to Option B. Not recommended over TanStack Query.

---

### 9.4 Option D: TanStack Query Core with RxJS Adapter

**Description:** Use TanStack Query Core (framework-agnostic) with a custom RxJS adapter. Leverage battle-tested SWR, deduplication, GC, and caching logic.

**Architecture:**

```
┌───────────────────────────────────────────────────────────────────────┐
│                          ViewModels                                    │
│  ┌─────────────────┐              ┌─────────────────┐                 │
│  │  ViewModel A    │              │  ViewModel B    │                 │
│  └────────┬────────┘              └────────┬────────┘                 │
└───────────┼────────────────────────────────┼──────────────────────────┘
            │                                │
            ▼                                ▼
┌───────────────────────────────────────────────────────────────────────┐
│                    RxJS Adapter Layer (~200 lines)                     │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  createQueryObservable(options): Observable<QueryResult>         │  │
│  │  createMutationObservable(options): [Observable, mutate]         │  │
│  │  getQueryDataSync(key): T | null                                 │  │
│  │  invalidateQueries(predicate): void                              │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌───────────────────────────────────────────────────────────────────────┐
│                    @tanstack/query-core (~13KB)                        │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  QueryClient (central orchestrator)                              │  │
│  │  ├─ QueryCache (Map<string, Query>)                             │  │
│  │  ├─ MutationCache (Set<Mutation>)                               │  │
│  │  └─ defaultOptions                                               │  │
│  │                                                                  │  │
│  │  QueryObserver (reactive observation)                            │  │
│  │  ├─ subscribe(callback)                                         │  │
│  │  ├─ getCurrentResult() ← SYNCHRONOUS!                           │  │
│  │  └─ trackResult() (property tracking optimization)              │  │
│  │                                                                  │  │
│  │  Built-in Features:                                              │  │
│  │  ├─ SWR (stale-while-revalidate)                                │  │
│  │  ├─ Request deduplication                                        │  │
│  │  ├─ Garbage collection                                          │  │
│  │  ├─ Retry with exponential backoff                              │  │
│  │  ├─ Window focus refetching                                      │  │
│  │  ├─ Network status refetching                                    │  │
│  │  └─ Hierarchical invalidation                                    │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
            │
            ▼ (optional)
┌───────────────────────────────────────────────────────────────────────┐
│             @tanstack/query-persist-client-core (~3KB)                 │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  Persister interface                                             │  │
│  │  Hydration/Dehydration                                           │  │
│  │  Selective persistence (shouldDehydrateQuery)                    │  │
│  │  Version-based invalidation                                      │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌───────────────────────────────────────────────────────────────────────┐
│                       AsyncStorage                                     │
└───────────────────────────────────────────────────────────────────────┘
```

**RxJS Adapter Implementation:**

```typescript
import { 
  QueryClient, 
  QueryObserver, 
  MutationObserver,
  QueryKey,
  QueryObserverOptions,
  MutationObserverOptions
} from '@tanstack/query-core';
import { Observable, BehaviorSubject } from 'rxjs';

// Singleton QueryClient
let queryClient: QueryClient | null = null;

export function getQueryClient(): QueryClient {
  if (!queryClient) {
    queryClient = new QueryClient({
      defaultOptions: {
        queries: {
          staleTime: 5 * 60 * 1000,      // 5 minutes
          gcTime: 10 * 60 * 1000,        // 10 minutes
          retry: 3,
          retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
        },
      },
    });
    queryClient.mount(); // Subscribe to focus/online managers
  }
  return queryClient;
}

/**
 * Creates an Observable wrapper around QueryObserver.
 * This is the primary API for ViewModels.
 */
export function createQueryObservable<TData, TError>(
  options: QueryObserverOptions<TData, TError>
): Observable<QueryObserverResult<TData, TError>> {
  return new Observable((subscriber) => {
    const client = getQueryClient();
    
    // Create observer
    const observer = new QueryObserver(client, options);
    
    // Get initial result SYNCHRONOUSLY
    const initialResult = observer.getCurrentResult();
    subscriber.next(initialResult);
    
    // Subscribe to changes
    const unsubscribe = observer.subscribe((result) => {
      subscriber.next(result);
    });
    
    // Cleanup
    return () => {
      unsubscribe();
      observer.destroy();
    };
  });
}

/**
 * Synchronous cache access for ViewModel constructors.
 */
export function getQueryDataSync<T>(queryKey: QueryKey): T | undefined {
  return getQueryClient().getQueryData<T>(queryKey);
}

/**
 * Set query data (for mutations).
 */
export function setQueryData<T>(queryKey: QueryKey, data: T): void {
  getQueryClient().setQueryData(queryKey, data);
}

/**
 * Invalidate queries.
 */
export function invalidateQueries(queryKey: QueryKey): Promise<void> {
  return getQueryClient().invalidateQueries({ queryKey });
}

/**
 * Creates a mutation observable.
 */
export function createMutationObservable<TData, TError, TVariables>(
  options: MutationObserverOptions<TData, TError, TVariables>
): {
  state$: Observable<MutationObserverResult<TData, TError, TVariables>>;
  mutate: (variables: TVariables) => void;
  mutateAsync: (variables: TVariables) => Promise<TData>;
} {
  const client = getQueryClient();
  const observer = new MutationObserver(client, options);
  
  const state$ = new Observable((subscriber) => {
    subscriber.next(observer.getCurrentResult());
    const unsubscribe = observer.subscribe((result) => {
      subscriber.next(result);
    });
    return unsubscribe;
  });
  
  return {
    state$,
    mutate: (variables) => observer.mutate(variables),
    mutateAsync: (variables) => observer.mutate(variables) as Promise<TData>,
  };
}
```

**ViewModel Integration:**

```typescript
class DeviceSettingsViewModel {
  private state: BehaviorSubject<DeviceSettingsState>;
  private subscription: Subscription;
  
  constructor() {
    // SYNCHRONOUS cache access - no loading flash!
    const cached = getQueryDataSync<DeviceSettings>(['device', 'settings']);
    
    this.state = new BehaviorSubject<DeviceSettingsState>({
      data: cached ?? null,
      loading: !cached,
      error: null,
    });
    
    // Subscribe to query updates
    this.subscription = createQueryObservable({
      queryKey: ['device', 'settings'],
      queryFn: () => this.api.getDeviceSettings(),
      staleTime: 5 * 60 * 1000,
    }).subscribe((result) => {
      this.state.next({
        data: result.data ?? null,
        loading: result.isLoading,
        error: result.error,
      });
    });
  }
  
  dispose(): void {
    this.subscription.unsubscribe();
  }
}
```

**What This Provides Out of the Box:**

| Feature | Built-In? | Configuration |
|---------|-----------|---------------|
| Synchronous cache access | ✅ | `getCurrentResult()`, `getQueryData()` |
| Cross-screen sync | ✅ | QueryObserver notifications |
| SWR | ✅ | `staleTime` option |
| Request deduplication | ✅ | Automatic |
| TTL/staleness | ✅ | `staleTime` option |
| Garbage collection | ✅ | `gcTime` option |
| Hierarchical invalidation | ✅ | Partial key matching |
| Retry with backoff | ✅ | `retry`, `retryDelay` options |
| Window focus refetch | ✅ | `refetchOnWindowFocus` option |
| Network reconnect refetch | ✅ | `refetchOnReconnect` option |
| Persistence | ✅ | `@tanstack/query-persist-client-core` |
| Selective persistence | ✅ | `shouldDehydrateQuery` |
| Mutation support | ✅ | `MutationObserver` |
| Optimistic updates | ✅ | `onMutate` callback |

**Effort Estimate:**
- RxJS Adapter: 1-2 days
- Integration with ViewModels: 1-2 days
- Persistence setup: 0.5-1 day
- Testing: 1-2 days
- **Total: 1-1.5 weeks**

**Pros:**
- Battle-tested by millions of apps
- All SWR, dedup, GC logic built-in
- Small bundle (~13KB)
- Active maintenance and community
- Extensive documentation
- TypeScript-first

**Cons:**
- External dependency
- Must learn TanStack Query concepts
- RxJS adapter adds thin layer

**Verdict:** ✅ **Recommended.** Best effort-to-value ratio.

---

## 10. Solution Comparison

### 10.1 Feature Matrix

| Feature | A: Storage | B: RxJS Custom | C: Zustand | D: TanStack |
|---------|------------|----------------|------------|-------------|
| **Sync cache access** | ❌ 60ms async | ✅ Instant | ✅ Instant | ✅ Instant |
| **Cross-screen sync** | ✅ Subscriptions | ✅ BehaviorSubject | ✅ Zustand sub | ✅ QueryObserver |
| **No loading flash** | ❌ Always flashes | ✅ | ✅ | ✅ |
| **SWR** | ❌ Build | ⚠️ Build (High) | ⚠️ Build (High) | ✅ Built-in |
| **Request dedup** | ❌ Impossible | ⚠️ Build (Med) | ⚠️ Build (Med) | ✅ Built-in |
| **TTL/Staleness** | ❌ Build | ⚠️ Build (Med) | ⚠️ Build (Med) | ✅ Built-in |
| **GC** | ❌ N/A | ⚠️ Build (Med) | ⚠️ Build (Med) | ✅ Built-in |
| **Hierarchical invalidation** | ⚠️ Build | ⚠️ Build (Med) | ⚠️ Build (Med) | ✅ Built-in |
| **Retry logic** | ❌ Build | ⚠️ Build (Med) | ⚠️ Build (Med) | ✅ Built-in |
| **Persistence** | ✅ Native | ⚠️ Build (High) | ✅ Middleware | ✅ Plugin |
| **RxJS native** | N/A | ✅ Native | ⚠️ Adapter | ⚠️ Adapter |
| **External dependency** | ✅ None | ✅ None | ⚠️ Zustand | ⚠️ TanStack |

### 10.2 Effort Matrix

| Aspect | A: Storage | B: RxJS Custom | C: Zustand | D: TanStack |
|--------|------------|----------------|------------|-------------|
| **Initial development** | 2-3 weeks | 3-4 weeks | 3-4 weeks | 1-1.5 weeks |
| **Ongoing maintenance** | Medium | High | High | Low |
| **Edge case discovery** | Ongoing | Ongoing | Ongoing | Already solved |
| **Documentation effort** | High | High | High | Low (use TanStack docs) |

### 10.3 Risk Matrix

| Risk | A: Storage | B: RxJS Custom | C: Zustand | D: TanStack |
|------|------------|----------------|------------|-------------|
| **Meets P0-1 (sync access)** | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes |
| **Unknown edge cases** | High | High | High | Low |
| **Performance issues** | Medium | Medium | Medium | Low |
| **Dependency risk** | None | None | Low | Low |

### 10.4 Decision Matrix

| Criteria | Weight | A: Storage | B: RxJS | C: Zustand | D: TanStack |
|----------|--------|------------|---------|------------|-------------|
| Meets P0 requirements | 30% | 3 | 9 | 9 | 10 |
| Development speed | 25% | 6 | 4 | 4 | 9 |
| Maintenance burden | 20% | 5 | 3 | 4 | 9 |
| Feature completeness | 15% | 4 | 6 | 6 | 10 |
| No external deps | 10% | 10 | 10 | 7 | 6 |
| **Weighted Score** | | **5.0** | **5.8** | **5.95** | **9.15** |

### 10.5 Recommendation

**TanStack Query Core (Option D)** is the clear winner:

1. **Only solution meeting all P0 requirements** with built-in features
2. **Fastest time to implementation** (1-1.5 weeks vs 3-4 weeks)
3. **Lowest maintenance burden** (library maintained by community)
4. **Battle-tested** (millions of production apps)
5. **Best documentation** (extensive guides and examples)
6. **Acceptable trade-off** (~13KB bundle, adapter layer needed)

---

## 11. Recommended Approach: TanStack Query Core

### 11.1 Why TanStack Query Core

TanStack Query is a framework-agnostic async state management library. The core package (`@tanstack/query-core`) contains all caching logic without any framework dependencies.

**Key Insight:** Official adapters exist for React, Vue, Solid, Svelte, and Angular—all are thin wrappers (~200-400 lines) around the core. We can build our own RxJS/MVVM adapter following the same pattern.

### 11.2 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Application Architecture                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                          UI Layer                                   │ │
│  │  ┌────────────────┐    ┌────────────────┐    ┌────────────────┐   │ │
│  │  │  Screen A      │    │  Screen B      │    │  Screen C      │   │ │
│  │  └───────┬────────┘    └───────┬────────┘    └───────┬────────┘   │ │
│  └──────────┼─────────────────────┼─────────────────────┼────────────┘ │
│             │                     │                     │              │
│  ┌──────────▼─────────────────────▼─────────────────────▼────────────┐ │
│  │                       ViewModel Layer                              │ │
│  │  ┌────────────────┐    ┌────────────────┐    ┌────────────────┐   │ │
│  │  │  ViewModel A   │    │  ViewModel B   │    │  ViewModel C   │   │ │
│  │  │  (BehaviorSubj)│    │  (BehaviorSubj)│    │  (BehaviorSubj)│   │ │
│  │  └───────┬────────┘    └───────┬────────┘    └───────┬────────┘   │ │
│  └──────────┼─────────────────────┼─────────────────────┼────────────┘ │
│             │                     │                     │              │
│  ┌──────────▼─────────────────────▼─────────────────────▼────────────┐ │
│  │                      RxJS Adapter Layer                            │ │
│  │  ┌────────────────────────────────────────────────────────────┐   │ │
│  │  │  createQueryObservable()     getQueryDataSync()             │   │ │
│  │  │  createMutationObservable()  invalidateQueries()            │   │ │
│  │  │  setQueryData()                                              │   │ │
│  │  └────────────────────────────────────────────────────────────┘   │ │
│  └───────────────────────────────────┬───────────────────────────────┘ │
│                                      │                                  │
│  ┌───────────────────────────────────▼───────────────────────────────┐ │
│  │                    TanStack Query Core                             │ │
│  │  ┌─────────────────────────────────────────────────────────────┐  │ │
│  │  │                     QueryClient                              │  │ │
│  │  │  ┌──────────────────────┐  ┌──────────────────────┐         │  │ │
│  │  │  │     QueryCache       │  │   MutationCache      │         │  │ │
│  │  │  │  ┌────┐ ┌────┐ ┌────┐│  │  ┌────┐ ┌────┐       │         │  │ │
│  │  │  │  │Q1  │ │Q2  │ │Q3  ││  │  │M1  │ │M2  │       │         │  │ │
│  │  │  │  └──┬─┘ └──┬─┘ └──┬─┘│  │  └────┘ └────┘       │         │  │ │
│  │  │  └────┼──────┼──────┼──┘  └──────────────────────┘         │  │ │
│  │  │       │      │      │                                       │  │ │
│  │  │  ┌────▼──────▼──────▼────────────────────────────────────┐ │  │ │
│  │  │  │                QueryObservers                          │ │  │ │
│  │  │  │  Observer A ──► Query 1                                │ │  │ │
│  │  │  │  Observer B ──► Query 1  (same query, both notified)  │ │  │ │
│  │  │  │  Observer C ──► Query 2                                │ │  │ │
│  │  │  └────────────────────────────────────────────────────────┘ │  │ │
│  │  └─────────────────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────┬───────────────────────────────┘ │
│                                      │ (optional)                       │
│  ┌───────────────────────────────────▼───────────────────────────────┐ │
│  │                    Persistence Layer                               │ │
│  │  ┌─────────────────────────────────────────────────────────────┐  │ │
│  │  │  Persister → AsyncStorage                                    │  │ │
│  │  │  Hydration on app start                                      │  │ │
│  │  │  Dehydration on cache change                                 │  │ │
│  │  └─────────────────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 11.3 The Three Principles of Building an Adapter

TanStack Query adapters follow three principles:

#### Principle 1: Create an Observer When Component is Created

Every ViewModel that needs query data creates a `QueryObserver` instance:

```typescript
class DeviceViewModel {
  private observer: QueryObserver;
  
  constructor() {
    this.observer = new QueryObserver(queryClient, {
      queryKey: ['device', 'settings'],
      queryFn: fetchDeviceSettings,
    });
  }
}
```

#### Principle 2: Subscribe to Changes in the Observer

The observer provides a `subscribe()` method:

```typescript
class DeviceViewModel {
  constructor() {
    // ...create observer...
    
    this.unsubscribe = this.observer.subscribe((result) => {
      // Called whenever query state changes
      this.state.next(result);
    });
  }
  
  dispose() {
    this.unsubscribe();
  }
}
```

#### Principle 3: Ensure Components Update When Changes Occur

For our MVVM architecture, this means updating the BehaviorSubject:

```typescript
this.observer.subscribe((result) => {
  this.state.next({
    data: result.data,
    loading: result.isLoading,
    error: result.error,
  });
});
```

### 11.4 Complete RxJS Adapter

```typescript
// file: core/data/query-adapter.ts

import {
  QueryClient,
  QueryObserver,
  MutationObserver,
  QueryKey,
  QueryObserverOptions,
  QueryObserverResult,
  MutationObserverOptions,
  MutationObserverResult,
  hashQueryKey,
} from '@tanstack/query-core';
import { Observable, BehaviorSubject, Subscription } from 'rxjs';

// ============================================================================
// QueryClient Singleton
// ============================================================================

let queryClient: QueryClient | null = null;

export function getQueryClient(): QueryClient {
  if (!queryClient) {
    queryClient = new QueryClient({
      defaultOptions: {
        queries: {
          staleTime: 5 * 60 * 1000,       // 5 minutes
          gcTime: 10 * 60 * 1000,         // 10 minutes
          retry: 3,
          retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 30000),
          refetchOnWindowFocus: true,
          refetchOnReconnect: true,
        },
      },
    });
    queryClient.mount();
  }
  return queryClient;
}

export function destroyQueryClient(): void {
  if (queryClient) {
    queryClient.unmount();
    queryClient.clear();
    queryClient = null;
  }
}

// ============================================================================
// Query Observable
// ============================================================================

export interface QueryOptions<TData, TError = Error> {
  queryKey: QueryKey;
  queryFn: () => Promise<TData>;
  staleTime?: number;
  gcTime?: number;
  enabled?: boolean;
  refetchOnWindowFocus?: boolean;
  refetchOnReconnect?: boolean;
  retry?: number | boolean;
}

export function createQueryObservable<TData, TError = Error>(
  options: QueryOptions<TData, TError>
): Observable<QueryObserverResult<TData, TError>> {
  return new Observable((subscriber) => {
    const client = getQueryClient();
    const observer = new QueryObserver<TData, TError>(client, options);
    
    // Emit initial result synchronously
    subscriber.next(observer.getCurrentResult());
    
    // Subscribe to updates
    const unsubscribe = observer.subscribe((result) => {
      subscriber.next(result);
    });
    
    // Cleanup
    return () => {
      unsubscribe();
      observer.destroy();
    };
  });
}

// ============================================================================
// Synchronous Cache Access
// ============================================================================

/**
 * Get cached data synchronously.
 * Use in ViewModel constructors to populate initial state.
 */
export function getQueryDataSync<T>(queryKey: QueryKey): T | undefined {
  return getQueryClient().getQueryData<T>(queryKey);
}

/**
 * Get full query state synchronously.
 */
export function getQueryStateSync(queryKey: QueryKey) {
  return getQueryClient().getQueryState(queryKey);
}

// ============================================================================
// Cache Manipulation
// ============================================================================

/**
 * Set query data directly (for optimistic updates).
 */
export function setQueryData<T>(
  queryKey: QueryKey,
  updater: T | ((oldData: T | undefined) => T)
): void {
  getQueryClient().setQueryData(queryKey, updater);
}

/**
 * Invalidate queries (mark as stale, trigger refetch if observed).
 */
export function invalidateQueries(
  queryKey: QueryKey,
  options?: { exact?: boolean }
): Promise<void> {
  return getQueryClient().invalidateQueries({
    queryKey,
    exact: options?.exact,
  });
}

/**
 * Invalidate all queries matching a predicate.
 */
export function invalidateQueriesMatching(
  predicate: (query: any) => boolean
): Promise<void> {
  return getQueryClient().invalidateQueries({ predicate });
}

/**
 * Remove queries from cache entirely.
 */
export function removeQueries(queryKey: QueryKey): void {
  getQueryClient().removeQueries({ queryKey });
}

// ============================================================================
// Mutation Observable
// ============================================================================

export interface MutationOptions<TData, TError = Error, TVariables = void> {
  mutationFn: (variables: TVariables) => Promise<TData>;
  onMutate?: (variables: TVariables) => Promise<unknown> | unknown;
  onSuccess?: (data: TData, variables: TVariables) => Promise<unknown> | unknown;
  onError?: (error: TError, variables: TVariables) => Promise<unknown> | unknown;
  onSettled?: (data: TData | undefined, error: TError | null, variables: TVariables) => Promise<unknown> | unknown;
}

export interface MutationResult<TData, TError, TVariables> {
  state$: Observable<MutationObserverResult<TData, TError, TVariables>>;
  mutate: (variables: TVariables) => void;
  mutateAsync: (variables: TVariables) => Promise<TData>;
  reset: () => void;
}

export function createMutationObservable<TData, TError = Error, TVariables = void>(
  options: MutationOptions<TData, TError, TVariables>
): MutationResult<TData, TError, TVariables> {
  const client = getQueryClient();
  const observer = new MutationObserver<TData, TError, TVariables>(client, options);
  
  const state$ = new Observable<MutationObserverResult<TData, TError, TVariables>>((subscriber) => {
    subscriber.next(observer.getCurrentResult());
    const unsubscribe = observer.subscribe((result) => {
      subscriber.next(result);
    });
    return unsubscribe;
  });
  
  return {
    state$,
    mutate: (variables) => observer.mutate(variables),
    mutateAsync: (variables) => observer.mutate(variables) as Promise<TData>,
    reset: () => observer.reset(),
  };
}

// ============================================================================
// Prefetching
// ============================================================================

/**
 * Prefetch data into cache without subscribing.
 */
export function prefetchQuery<TData>(
  options: QueryOptions<TData>
): Promise<void> {
  return getQueryClient().prefetchQuery(options);
}

// ============================================================================
// ViewModel Helper
// ============================================================================

/**
 * Helper class for ViewModels to manage query subscriptions.
 */
export class QuerySubscriptionManager {
  private subscriptions: Subscription[] = [];
  
  subscribe<T>(
    observable: Observable<T>,
    next: (value: T) => void
  ): void {
    this.subscriptions.push(observable.subscribe(next));
  }
  
  dispose(): void {
    this.subscriptions.forEach(sub => sub.unsubscribe());
    this.subscriptions = [];
  }
}
```

### 11.5 ViewModel Integration Pattern

```typescript
// file: features/device/DeviceSettingsViewModel.ts

import { BehaviorSubject, Subscription } from 'rxjs';
import {
  createQueryObservable,
  createMutationObservable,
  getQueryDataSync,
  setQueryData,
  invalidateQueries,
} from 'core/data/query-adapter';

interface DeviceSettingsState {
  data: DeviceSettings | null;
  loading: boolean;
  error: Error | null;
  isSaving: boolean;
}

export class DeviceSettingsViewModel {
  private state: BehaviorSubject<DeviceSettingsState>;
  private subscription: Subscription;
  private saveMutation;
  
  constructor(private deviceId: string, private api: DeviceApi) {
    // =========================================================================
    // CRITICAL: Synchronous cache access for initial state
    // This is what eliminates the loading flash!
    // =========================================================================
    const cached = getQueryDataSync<DeviceSettings>(['device', deviceId, 'settings']);
    
    this.state = new BehaviorSubject<DeviceSettingsState>({
      data: cached ?? null,
      loading: !cached,
      error: null,
      isSaving: false,
    });
    
    // Subscribe to query updates
    this.subscription = createQueryObservable({
      queryKey: ['device', deviceId, 'settings'],
      queryFn: () => this.api.getDeviceSettings(deviceId),
      staleTime: 5 * 60 * 1000, // 5 minutes
    }).subscribe((result) => {
      this.state.next({
        ...this.state.getValue(),
        data: result.data ?? null,
        loading: result.isLoading,
        error: result.error ?? null,
      });
    });
    
    // Setup save mutation
    this.saveMutation = createMutationObservable({
      mutationFn: (settings: DeviceSettings) => 
        this.api.saveDeviceSettings(deviceId, settings),
      onSuccess: (data) => {
        // Update cache - all observers will be notified!
        setQueryData(['device', deviceId, 'settings'], data);
      },
    });
    
    this.saveMutation.state$.subscribe((result) => {
      this.state.next({
        ...this.state.getValue(),
        isSaving: result.isPending,
      });
    });
  }
  
  // Public API
  get state$() {
    return this.state.asObservable();
  }
  
  async saveSettings(settings: DeviceSettings): Promise<void> {
    await this.saveMutation.mutateAsync(settings);
  }
  
  refresh(): void {
    invalidateQueries(['device', this.deviceId, 'settings']);
  }
  
  dispose(): void {
    this.subscription.unsubscribe();
  }
}
```

### 11.6 Cross-Screen Synchronization

This is how updates flow across screens automatically:

```
Cross-Screen Sync Flow:
─────────────────────────────────────────────────────────────────────────

Screen A (Alexa Page)           Screen B (Voice Selection)
        │                               │
        │ Observer A                    │ Observer B
        │    │                          │    │
        └────┼──────────────────────────┼────┘
             │                          │
             ▼                          ▼
    ┌─────────────────────────────────────────────────┐
    │           Query: ['voice', 'settings']          │
    │                                                 │
    │  observers: [Observer A, Observer B]            │
    │  data: { currentVoice: 'Default' }              │
    │                                                 │
    └─────────────────────────────────────────────────┘
                           │
                           │ User selects "Voice 3" on Screen B
                           │
                           ▼
    ┌─────────────────────────────────────────────────┐
    │           Mutation: saveVoiceSetting            │
    │                                                 │
    │  onSuccess: setQueryData(                       │
    │    ['voice', 'settings'],                       │
    │    { currentVoice: 'Voice 3' }                  │
    │  )                                              │
    │                                                 │
    └─────────────────────────────────────────────────┘
                           │
                           │ Cache updated
                           │
                           ▼
    ┌─────────────────────────────────────────────────┐
    │           Query: ['voice', 'settings']          │
    │                                                 │
    │  data: { currentVoice: 'Voice 3' }  ← UPDATED  │
    │                                                 │
    │  Notify all observers!                          │
    │    → Observer A notified                        │
    │    → Observer B notified                        │
    │                                                 │
    └─────────────────────────────────────────────────┘
                           │
            ┌──────────────┴──────────────┐
            │                             │
            ▼                             ▼
    Screen A updates                Screen B updates
    Shows "Voice 3"                 Shows "Voice 3"
    
    NO API CALL NEEDED!
    NO LOADING STATE!
    INSTANT UPDATE!

─────────────────────────────────────────────────────────────────────────
```

### 11.7 Memory Footprint Analysis

```
Memory Usage Breakdown:
─────────────────────────────────────────────────────────────────────────

TanStack Query Core:
  Library code:                 ~13 KB (gzipped: ~4 KB)
  Runtime overhead:             ~2-3 KB

Cache Data (typical Settings app):
  Device Settings:              ~500 bytes
  User Preferences:             ~300 bytes
  Voice Settings:               ~200 bytes
  Available Voices List:        ~3 KB
  Device Info:                  ~500 bytes
  ─────────────────────────────
  Total cached data:            ~5 KB

Query Metadata:
  Per query overhead:           ~200 bytes
  With 10 active queries:       ~2 KB

Observer Overhead:
  Per observer:                 ~100 bytes
  With 5 active observers:      ~500 bytes

─────────────────────────────────────────────────────────────────────────

TOTAL MEMORY IMPACT:            ~25 KB

Context:
  Typical RN app memory:        100-200 MB
  Cache as % of app:            0.01-0.025%
  One medium image:             ~50 KB (2x our cache)

─────────────────────────────────────────────────────────────────────────
```

### 11.8 Persistence Configuration

```typescript
// file: core/data/query-persistence.ts

import { createAsyncStoragePersister } from '@tanstack/query-async-storage-persister';
import { persistQueryClient } from '@tanstack/query-persist-client-core';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { getQueryClient } from './query-adapter';

const CACHE_VERSION = '1.0.0';

export async function setupQueryPersistence(): Promise<void> {
  const persister = createAsyncStoragePersister({
    storage: AsyncStorage,
    key: `SETTINGS_QUERY_CACHE_${CACHE_VERSION}`,
  });
  
  await persistQueryClient({
    queryClient: getQueryClient(),
    persister,
    maxAge: 12 * 60 * 60 * 1000, // 12 hours
    dehydrateOptions: {
      shouldDehydrateQuery: (query) => {
        // Only persist successful queries
        if (query.state.status !== 'success') return false;
        
        // Only persist queries marked for persistence
        return query.meta?.persist === true;
      },
    },
  });
}

// Mark queries for persistence
export const QUERY_DEFAULTS = {
  deviceSettings: {
    staleTime: 5 * 60 * 1000,
    meta: { persist: true },
  },
  voiceSettings: {
    staleTime: 5 * 60 * 1000,
    meta: { persist: true },
  },
  availableVoices: {
    staleTime: 60 * 60 * 1000, // 1 hour
    meta: { persist: true },
  },
  // Critical data - never persist
  deviceRegistration: {
    staleTime: 0,
    meta: { persist: false },
  },
};
```

### 11.9 External Invalidation Strategy

For the critical scenario where external events require cache invalidation:

```typescript
// file: background/cache-invalidation-task.ts

import AsyncStorage from '@react-native-async-storage/async-storage';

const CACHE_KEY = 'SETTINGS_QUERY_CACHE_1.0.0';
const VALIDATION_TOKEN_KEY = 'CACHE_VALIDATION_TOKEN';

/**
 * Run in background task when push notification received
 * or periodically to check for invalidation conditions.
 */
export async function runCacheInvalidationCheck(): Promise<void> {
  try {
    // Check if invalidation is needed
    const shouldInvalidate = await checkInvalidationCondition();
    
    if (shouldInvalidate) {
      // Option 1: Clear entire cache
      await AsyncStorage.removeItem(CACHE_KEY);
      
      // Option 2: Update validation token (foreground will detect mismatch)
      await AsyncStorage.setItem(VALIDATION_TOKEN_KEY, Date.now().toString());
      
      console.log('Cache invalidated by background task');
    }
  } catch (error) {
    console.error('Cache invalidation check failed:', error);
  }
}

async function checkInvalidationCondition(): Promise<boolean> {
  // Example: Check if device is still registered
  const response = await fetch('/api/device/status', {
    method: 'HEAD',
  });
  
  return response.status === 404; // Device no longer exists
}
```

```typescript
// file: app/startup.ts

import AsyncStorage from '@react-native-async-storage/async-storage';
import { getQueryClient } from 'core/data/query-adapter';

const VALIDATION_TOKEN_KEY = 'CACHE_VALIDATION_TOKEN';

/**
 * Run during app startup, before hydrating cache.
 */
export async function validateCacheBeforeHydration(): Promise<boolean> {
  try {
    // Get stored validation token
    const storedToken = await AsyncStorage.getItem(VALIDATION_TOKEN_KEY);
    
    // Get current token from server
    const response = await fetch('/api/cache/validation-token');
    const { token: serverToken } = await response.json();
    
    if (storedToken !== serverToken) {
      // Cache is invalid - clear it
      await AsyncStorage.removeItem('SETTINGS_QUERY_CACHE_1.0.0');
      await AsyncStorage.setItem(VALIDATION_TOKEN_KEY, serverToken);
      return false; // Cache was invalidated
    }
    
    return true; // Cache is valid
  } catch (error) {
    // On error, be conservative - invalidate cache
    await AsyncStorage.removeItem('SETTINGS_QUERY_CACHE_1.0.0');
    return false;
  }
}
```

### 11.10 TanStack Query Core in Background Tasks

**Confirmation:** TanStack Query Core works in background tasks.

```
TanStack Query Core Runtime Requirements:
─────────────────────────────────────────────────────────────────────────

REQUIRES:
  ✓ Map, Set, WeakMap (ES6 built-ins)
  ✓ Promise (ES6)
  ✓ setTimeout, clearTimeout
  ✓ Date.now()
  ✓ console (optional, for logging)

DOES NOT REQUIRE:
  ✗ window
  ✗ document
  ✗ DOM
  ✗ React
  ✗ Any UI framework

WORKS IN:
  ✓ Node.js
  ✓ React Native main thread
  ✓ React Native background tasks (Headless JS)
  ✓ Web Workers
  ✓ Service Workers
  ✓ Any JavaScript runtime

─────────────────────────────────────────────────────────────────────────
```

**Background Task Usage Example:**

```typescript
// file: background/headless-task.ts

import { QueryClient } from '@tanstack/query-core';

// This runs in Headless JS (no UI)
export async function backgroundCacheTask(): Promise<void> {
  // Create standalone QueryClient for background
  const bgClient = new QueryClient();
  
  try {
    // Perform cache operations
    await bgClient.prefetchQuery({
      queryKey: ['device', 'settings'],
      queryFn: () => fetch('/api/device/settings').then(r => r.json()),
    });
    
    // Serialize and persist
    const cache = bgClient.getQueryCache().getAll();
    // ... persist to AsyncStorage
    
  } finally {
    bgClient.clear();
  }
}
```

---

## 12. Implementation Plan

### 12.1 Phase 1: Core Infrastructure (Week 1)

| Day | Task | Deliverable |
|-----|------|-------------|
| 1-2 | Create RxJS adapter | `query-adapter.ts` with all functions |
| 2-3 | Setup QueryClient singleton | Proper initialization, mount/unmount |
| 3-4 | Create ViewModel base class | `QuerySubscriptionManager`, helpers |
| 4-5 | Unit tests | Adapter tests, mock API tests |

### 12.2 Phase 2: Migration - First Use Case (Week 2)

| Day | Task | Deliverable |
|-----|------|-------------|
| 1-2 | Migrate Alexa Voice flow | Updated ViewModels |
| 2-3 | Test cross-screen sync | Verify mutation → observer flow |
| 3-4 | Remove focus listeners | Replace with cache-based sync |
| 4-5 | Integration testing | End-to-end flow testing |

### 12.3 Phase 3: Migration - Second Use Case (Week 2-3)

| Day | Task | Deliverable |
|-----|------|-------------|
| 1-2 | Migrate Device Location flow | Updated ViewModels |
| 2-3 | Add mutation with optimistic update | Instant feedback |
| 3 | Test edge cases | Error handling, retry, etc. |

### 12.4 Phase 4: Persistence & Polish (Week 3)

| Day | Task | Deliverable |
|-----|------|-------------|
| 1-2 | Add persistence layer | Cold start optimization |
| 2-3 | External invalidation | Background task integration |
| 3-4 | Performance validation | No regressions |
| 4-5 | Documentation | Usage guide for team |

### 12.5 Success Metrics

| Metric | Current | Target | How to Measure |
|--------|---------|--------|----------------|
| Loading states on back nav | Every time | Never (cached) | Manual testing |
| Duplicate API calls | Common | Zero | Network logging |
| Time to display cached data | 500ms+ | <10ms | Performance tracing |
| Memory increase | N/A | <30KB | Memory profiling |
| Bundle size increase | N/A | <15KB | Bundle analysis |

---

## 13. Risks and Mitigations

### 13.1 Risk: Learning Curve

**Risk:** Team unfamiliar with TanStack Query concepts.

**Impact:** Medium

**Mitigations:**
- Adapter abstracts most complexity
- Create internal documentation with examples
- Patterns similar to existing RxJS usage
- Pair programming during initial migration
- TanStack Query has excellent documentation

### 13.2 Risk: Bundle Size

**Risk:** Adding ~13KB to bundle.

**Impact:** Low

**Mitigations:**
- 13KB is minimal compared to typical RN bundles (2-5MB)
- Eliminates need to build equivalent functionality
- Tree-shaking removes unused code
- Gzipped size is only ~4KB

### 13.3 Risk: Integration Issues

**Risk:** Unforeseen issues integrating with MVVM pattern.

**Impact:** Medium

**Mitigations:**
- Proof of concept before full implementation
- Adapter provides clean abstraction layer
- Can extend adapter for edge cases
- Fallback: Can always customize further

### 13.4 Risk: Cold Start Flash

**Risk:** Even with persistence, cold start still has async storage read.

**Impact:** Medium

**Mitigations:**
- App-level hydration gate during splash screen
- Hydration completes before main UI renders
- Same behavior as current app, no regression
- Use validation token pattern for fast checks

### 13.5 Risk: External Invalidation Complexity

**Risk:** Background task may not run reliably.

**Impact:** Medium

**Mitigations:**
- Conservative cache policy for critical data (don't persist)
- Startup validation as fallback
- Version-based cache keys
- Multiple layers of protection

### 13.6 Risk: Memory Pressure

**Risk:** Caching too much data causes memory issues.

**Impact:** Low

**Mitigations:**
- gcTime ensures unused queries are cleaned up
- Typical cache size is ~5-10KB
- Configure aggressive GC for constrained devices
- Monitor memory in production

---

## 14. Appendices

### 14.1 Appendix A: Glossary

| Term | Definition |
|------|------------|
| **QueryClient** | Central orchestrator that manages caches and defaults |
| **QueryCache** | In-memory Map storing Query instances |
| **Query** | Individual cached query with state and lifecycle |
| **QueryObserver** | Reactive observer that subscribes to Query changes |
| **Mutation** | Operation that modifies server state |
| **staleTime** | Duration after which data is considered stale |
| **gcTime** | Duration after which unused queries are garbage collected |
| **SWR** | Stale-While-Revalidate: show stale data while refreshing |
| **Hydration** | Restoring cache from persistent storage |
| **Dehydration** | Serializing cache to persistent storage |

### 14.2 Appendix B: Requirements Traceability

| Requirement | TanStack Query Feature | Notes |
|-------------|------------------------|-------|
| P0-1: Sync Cache Access | `getCurrentResult()`, `getQueryData()` | Core feature |
| P0-2: Cross-Screen Sync | QueryObserver notification system | Core feature |
| P0-3: SWR | Built-in with `staleTime` | Core feature |
| P0-4: TTL | `staleTime` option per query | Core feature |
| P0-5: Manual Invalidation | `invalidateQueries()` method | Core feature |
| P0-6: RxJS Compatibility | Custom adapter layer | ~200 lines |
| P0-7: Mutation Support | `MutationObserver` + `setQueryData` | Core feature |
| P1-1: Request Deduplication | Automatic for same queryKey | Core feature |
| P1-2: Garbage Collection | `gcTime` option | Core feature |
| P1-3: Hierarchical Invalidation | Partial key matching | Core feature |
| P1-4: Persistence | `@tanstack/query-persist-client-core` | Official plugin |
| P1-5: Refetch Coordination | Single in-flight request per queryKey | Core feature |
| P1-6: External Invalidation | Custom via storage + validation | Implementation needed |

### 14.3 Appendix C: Alternative Solutions Considered

| Solution | Why Not Recommended |
|----------|---------------------|
| Storage-Only | Cannot provide sync cache access (fails P0-1) |
| Custom RxJS Cache | High effort (3-4 weeks), maintenance burden |
| Zustand + Custom SWR | Same effort as custom, adds dependency |
| Redux + RTK Query | Heavy bundle (~50KB+), overkill |
| SWR (Vercel) | React-specific, would need adapter anyway |
| Apollo Client | GraphQL-focused, heavy for REST |

### 14.4 Appendix D: Bundle Size Comparison

| Library | Bundle Size | Gzipped |
|---------|-------------|---------|
| @tanstack/query-core | ~13KB | ~4KB |
| Zustand | ~1KB | ~0.5KB |
| Redux + RTK | ~25KB | ~8KB |
| Apollo Client | ~50KB | ~15KB |
| SWR | ~12KB | ~4KB |

### 14.5 Appendix E: References

- [TanStack Query Documentation](https://tanstack.com/query)
- [TanStack Query Core Source](https://github.com/TanStack/query/tree/main/packages/query-core)
- [Building an Adapter (query.gg)](https://query.gg/building-an-adapter)
- [Persistence Documentation](https://tanstack.com/query/latest/docs/framework/react/plugins/persistQueryClient)

---

*End of Design Document*
