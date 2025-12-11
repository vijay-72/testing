# Settings App: Data Caching Layer Design

**Author:** [Your Name]  
**Date:** December 2024  
**Status:** Draft - For Design Review

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Statement](#2-problem-statement)
3. [Requirements](#3-requirements)
4. [Technical Context](#4-technical-context)
5. [What SWR Actually Requires](#5-what-swr-actually-requires)
6. [Option A: DS-Only Solution](#6-option-a-ds-only-solution)
7. [Option B: TanStack Query Core](#7-option-b-tanstack-query-core)
8. [Edge Cases Deep Dive](#8-edge-cases-deep-dive)
9. [Comparison Matrix](#9-comparison-matrix)
10. [Risk Analysis](#10-risk-analysis)
11. [Recommendation](#11-recommendation)
12. [Implementation Plan](#12-implementation-plan)
13. [Open Questions](#13-open-questions)

---

## 1. Executive Summary

### The Problem
Users see unnecessary loading indicators when navigating between screens in the Settings app. Every screen visit triggers a fresh API call (200-500ms) even when:
- User just viewed this screen seconds ago
- User made a change in an inner screen (we already know the new value)
- Data hasn't changed on the backend

### The Solution
Implement a caching layer with Stale-While-Revalidate (SWR) pattern: show cached data immediately, refresh in background.

### Two Viable Approaches

| Approach | Summary | Effort |
|----------|---------|--------|
| **DS-Only** | Use existing DataStore (persistent storage) as cache, build SWR logic ourselves | ~1.5-2 weeks |
| **TanStack Query Core** | Use battle-tested caching library with custom RxJS adapter | ~1 week |

### Key Trade-off
**DS-Only:** No external dependencies, but we own SWR complexity (retry, dedup, error states)  
**TanStack Query:** External dependency, but SWR complexity is handled by proven library

---

## 2. Problem Statement

### 2.1 Current User Experience

```
┌─────────────────────────────────────────────────────────────────────┐
│ CURRENT FLOW: User navigates to Alexa Page                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  T=0ms      User taps "Alexa Settings"                              │
│  T=10ms     Screen mounts, ViewModel.initialize() called            │
│  T=11ms     API call starts: GET /voice-settings                    │
│  T=11ms     UI shows: [████████████] (skeleton loader)              │
│             User waits...                                            │
│  T=300ms    API returns data                                         │
│  T=301ms    UI shows: Voice: Default                                │
│                                                                      │
│  User sees loading for ~290ms                                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ CURRENT FLOW: User changes voice and goes back                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  T=0ms      User on Alexa Page, sees "Voice: Default"               │
│  T=100ms    User taps Voice setting                                 │
│  T=200ms    Voice Selection screen opens                            │
│  T=5000ms   User selects "Voice 3"                                  │
│  T=5300ms   API saves successfully                                  │
│  T=5500ms   User presses BACK                                       │
│                                                                      │
│  T=5501ms   Alexa Page regains focus                                │
│  T=5502ms   performRefresh() → API call starts                      │
│  T=5502ms   UI shows: [████████████] (skeleton AGAIN!)              │
│             User: "Why loading? I just changed it!"                  │
│  T=5800ms   API returns "Voice 3"                                   │
│  T=5801ms   UI shows: Voice: Voice 3                                │
│                                                                      │
│  User sees UNNECESSARY loading for ~300ms                            │
│  We KNEW it was "Voice 3" - we just saved it!                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 The Customer Complaint

> "Every time I open a screen, I see a loading indicator. When I step into an inner screen and come back, I didn't even make a change to that setting but it still shows a loading indicator again."

### 2.3 Root Causes

1. **No caching:** API responses are not stored for reuse
2. **No cross-screen communication:** When inner screen updates data, outer screen doesn't know
3. **Defensive refetching:** On every focus, we refetch because we don't know if data changed

---

## 3. Requirements

### 3.1 P0 — Must Have

| ID | Requirement | Description |
|----|-------------|-------------|
| P0-1 | **Cache API responses** | Store responses for reuse across screen visits |
| P0-2 | **Persist across app death** | Cache must survive app being killed by OS |
| P0-3 | **Cross-screen sync** | When Screen B updates data, Screen A sees it without refetching |
| P0-4 | **Eliminate unnecessary loading** | No loading indicator for cached data |
| P0-5 | **Stale-While-Revalidate** | Show cached data immediately, refresh in background if stale |
| P0-6 | **Cache invalidation** | Ability to mark cache invalid (e.g., on background event) |
| P0-7 | **Fit existing architecture** | Work with BaseViewModel, BehaviorSubject, RxJS patterns |

### 3.2 P1 — Should Have

| ID | Requirement | Description |
|----|-------------|-------------|
| P1-1 | **Optimistic updates** | Update UI before API confirms, rollback on error |
| P1-2 | **Error handling** | Graceful handling when background refresh fails |
| P1-3 | **Retry logic** | Automatic retry on transient failures |

### 3.3 P2 — Nice to Have

| ID | Requirement | Description |
|----|-------------|-------------|
| P2-1 | **Request deduplication** | Don't make duplicate API calls for same data |
| P2-2 | **Configurable staleness** | Different stale times per query type |
| P2-3 | **Rich status states** | Track isLoading, isFetching, isStale, error per query |

---

## 4. Technical Context

### 4.1 Current Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         React Native App                             │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Screen (extends BaseScreen)                                          │
│   • Gets ViewModel from factory                                      │
│   • useObservable() subscribes to BehaviorSubject                   │
│   • onFocus listener calls viewModel.performRefresh()               │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ ViewModel (extends BaseViewModel)                                    │
│                                                                      │
│   constructor() {                                                    │
│     super({ voice: '', loading: true, ... });                       │
│   }                                                                  │
│                                                                      │
│   initialize() {                                                     │
│     // Called after mount                                            │
│     this.fetchVoiceSettings();  // Always hits API                  │
│   }                                                                  │
│                                                                      │
│   performRefresh() {                                                 │
│     // Called on screen focus                                        │
│     this.fetchVoiceSettings();  // Hits API again                   │
│   }                                                                  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ BaseViewModel                                                        │
│   • Holds BehaviorSubject<State>                                    │
│   • Provides updateState(), getState()                              │
│   • Child ViewModels extend this                                    │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 DataStore (DS) - Persistent Storage

The app has an existing persistent storage system called DataStore (DS):

```
┌─────────────────────────────────────────────────────────────────────┐
│ DataStore (DS)                                                       │
├─────────────────────────────────────────────────────────────────────┤
│ • Technology: On-device process via IPC                             │
│ • Exposed via: TurboModule                                          │
│ • API: get(key), set(key, value), subscribe(key)                   │
│                                                                      │
│ Performance (benchmarked):                                           │
│   • Read: 60-70ms                                                   │
│   • Write: ~50ms                                                    │
│                                                                      │
│ Event System:                                                        │
│   • On write: publishes event to EventBus                          │
│   • Event contains: FULL VALUE (not just key)                       │
│   • Delivery: Async, very fast after write                          │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.3 EventBus Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│ EventBus (Singleton Service)                                         │
├─────────────────────────────────────────────────────────────────────┤
│ • Has internal BehaviorSubject                                      │
│ • publish(event): calls .next() on subject                          │
│ • subscribe(filter): returns filtered Observable                    │
│                                                                      │
│ Pausable Observable Pattern:                                         │
│   • ViewModels wrap EventBus subscription in pausable observable    │
│   • When screen blurs: stops processing events                      │
│   • Events still emit, just not processed                           │
│   • CAN BE CONFIGURED per event type                                │
│   • DS events can be configured to NOT pause                        │
└─────────────────────────────────────────────────────────────────────┘

Usage:
┌──────────────┐     publishes      ┌──────────────┐
│  DataStore   │ ─────────────────▶ │   EventBus   │
└──────────────┘                    └──────────────┘
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    │                      │                      │
                    ▼                      ▼                      ▼
            ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
            │ ViewModel A  │      │ ViewModel B  │      │ ViewModel C  │
            │ (subscribed) │      │ (subscribed) │      │  (paused)    │
            └──────────────┘      └──────────────┘      └──────────────┘
```

### 4.4 Current Loading Pattern

For IPC-based data, the app already uses a "wait for all, then show" pattern:

```typescript
// Current pattern for IPC data
class SomeViewModel extends BaseViewModel {
  constructor() {
    super({ loading: true, ...initialState });
  }
  
  async initialize() {
    // Load ALL IPC data in parallel
    const [data1, data2, data3] = await Promise.all([
      ipcService.getData1(),
      ipcService.getData2(), 
      ipcService.getData3(),
    ]);
    
    // Only THEN remove loading state
    this.updateState({
      loading: false,
      data1,
      data2,
      data3,
    });
  }
}

// Result: 
// - Skeleton shown during load (~200ms)
// - Then full screen with all data
// - No individual "retrieving" states per option
```

**Key insight:** If voice/address APIs move to DS with this pattern, they'd be part of the initial load. User sees skeleton for ~200ms (maybe +60-70ms), then full screen. This is acceptable.

### 4.5 Key Timings

| Operation | Latency | Notes |
|-----------|---------|-------|
| API call (GET) | 200-500ms | Network dependent |
| API call (POST/PUT) | 200-500ms | Network dependent |
| DS read | 60-70ms | Benchmarked |
| DS write | ~50ms | Benchmarked |
| DS event delivery | <10ms | After write completes |
| In-memory Map read | <1ms | Negligible |

### 4.6 Scale

| Metric | Value |
|--------|-------|
| Number of API endpoints to cache | ~10 |
| Average response size | ~1-2KB |
| Total cacheable data | ~15-20KB |

---

## 5. What SWR Actually Requires

Stale-While-Revalidate sounds simple: "show cached, refresh in background." But a robust implementation requires handling several concerns:

### 5.1 Core SWR Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SWR Implementation Components                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. STALENESS TRACKING                                              │
│     • Store timestamp with data: { value, fetchedAt }               │
│     • Calculate: isStale = (now - fetchedAt) > staleTime            │
│     • Decide: should we refresh?                                    │
│                                                                      │
│  2. BACKGROUND REFRESH                                              │
│     • Fetch fresh data WITHOUT showing loading                      │
│     • Update cache when complete                                    │
│     • Don't block user interaction                                  │
│                                                                      │
│  3. ERROR HANDLING                                                  │
│     • What if background refresh fails?                             │
│     • Keep showing stale data? (usually yes)                        │
│     • Track error state?                                            │
│     • Retry? When? How many times?                                  │
│                                                                      │
│  4. CACHE UPDATES + NOTIFICATIONS                                   │
│     • When cache updates, who needs to know?                        │
│     • Multiple ViewModels watching same key                         │
│     • Sync vs async notification                                    │
│                                                                      │
│  5. REQUEST COORDINATION                                            │
│     • What if refresh already in progress?                          │
│     • Don't start duplicate requests                                │
│     • Share in-flight promise                                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 Edge Cases to Handle

| Edge Case | Description | Required Handling |
|-----------|-------------|-------------------|
| Double refresh | User navigates away and back while refresh in progress | Deduplicate requests |
| Error during refresh | Network fails during background refresh | Keep stale data, maybe retry |
| Stale on error | Data is stale, refresh keeps failing | Track error state, limit retries |
| Race condition | Multiple ViewModels trigger refresh for same key | Single in-flight request |
| Optimistic rollback | Optimistic update, but API fails | Restore previous value |
| Rapid navigation | User quickly navigates between screens | Cancel or ignore stale requests |
| Cache invalidation | External event requires fresh data | Clear cache, force refresh |

### 5.3 SWR Complexity Spectrum

```
SIMPLE                                                           COMPLEX
   │                                                                 │
   ▼                                                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│ Just cache    Staleness    Background    Retry    Dedup    Full     │
│ + show        tracking     refresh       logic    requests  state   │
│                                                             machine │
├──────────────────────────────────────────────────────────────────────┤
│ ~2 days       +1 day       +1 day        +1 day   +1 day   +2 days  │
└──────────────────────────────────────────────────────────────────────┘

Minimum viable: Cache + Staleness + Background refresh = ~4-5 days
Robust solution: + Retry + Dedup + State tracking = ~7-10 days
```

---

## 6. Option A: DS-Only Solution

### 6.1 Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                            ViewModel                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  constructor() {                                                     │
│    super({ loading: true, voice: '' });                             │
│  }                                                                   │
│                                                                      │
│  async initialize() {                                                │
│    // Subscribe to DS events (don't pause for DS events)            │
│    eventBus.subscribe('ds:voice-settings')                          │
│      .subscribe(event => {                                          │
│        this.updateState({ voice: event.value.voice });              │
│      });                                                             │
│                                                                      │
│    // Read from DS                                                   │
│    const cached = await ds.get('voice-settings'); // 60-70ms        │
│    if (cached) {                                                     │
│      this.updateState({ voice: cached.value, loading: false });     │
│    }                                                                 │
│                                                                      │
│    // Background refresh if stale                                    │
│    if (!cached || this.isStale(cached)) {                           │
│      this.refreshInBackground('voice-settings');                    │
│    }                                                                 │
│  }                                                                   │
│                                                                      │
│  async refreshInBackground(key: string) {                           │
│    try {                                                             │
│      const fresh = await api.getVoiceSettings();                    │
│      await ds.set(key, {                                            │
│        value: fresh,                                                │
│        fetchedAt: Date.now()                                        │
│      });                                                             │
│      // DS event will notify all subscribers                        │
│    } catch (error) {                                                 │
│      // Handle error (see Section 6.5)                              │
│    }                                                                 │
│  }                                                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     DataStore (Persistent)                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  {                                                                   │
│    'voice-settings': {                                              │
│      value: { voice: 'Default', ... },                              │
│      fetchedAt: 1701234567890                                       │
│    },                                                                │
│    'device-settings': { ... },                                      │
│    ...                                                               │
│  }                                                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ (on write, publishes event with full value)
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                           EventBus                                   │
│                                                                      │
│  Event: { type: 'ds:voice-settings', value: { voice: 'Voice 3' } } │
│                                                                      │
│  → All subscribed ViewModels receive this immediately               │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 Cross-Screen Sync Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│ SCENARIO: User changes voice in inner screen, goes back             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  T=0ms      Alexa Page showing "Voice: Default"                     │
│             Subscribed to ds:voice-settings events (NOT paused)     │
│                                                                      │
│  T=100ms    User taps → Voice Selection opens                       │
│             Alexa Page subscription: still active for DS events     │
│                                                                      │
│  T=5000ms   User selects "Voice 3"                                  │
│             Voice Selection calls API...                            │
│                                                                      │
│  T=5300ms   API returns success                                     │
│             Voice Selection writes to DS:                           │
│               ds.set('voice-settings', { voice: 'Voice 3' })        │
│                                                                      │
│  T=5350ms   DS write completes                                      │
│             DS publishes event: { type: 'ds:voice-settings',        │
│                                   value: { voice: 'Voice 3' } }     │
│                                                                      │
│  T=5351ms   EventBus delivers to Alexa Page                         │
│             Alexa Page updates BehaviorSubject                      │
│             (User hasn't even pressed BACK yet!)                    │
│                                                                      │
│  T=5500ms   User presses BACK                                       │
│                                                                      │
│  T=5501ms   Alexa Page visible                                      │
│             Already showing "Voice 3" ✓                             │
│             No loading, no flash, correct data ✓                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

RESULT: Cross-screen sync works via DS events
```

### 6.3 Integration with Existing Loading Pattern

Voice/address settings would join the existing "load all, then show" pattern:

```typescript
async initialize() {
  // Existing IPC data + NEW DS cache reads
  const [
    existingData1,
    existingData2,
    voiceSettings,    // NEW: from DS
    addressSettings,  // NEW: from DS
  ] = await Promise.all([
    ipcService.getData1(),
    ipcService.getData2(),
    ds.get('voice-settings'),    // 60-70ms
    ds.get('address-settings'),  // 60-70ms (parallel, so not additive)
  ]);
  
  // Remove loading state
  this.updateState({
    loading: false,
    existingData1,
    existingData2,
    voice: voiceSettings?.value?.voice ?? '',
    address: addressSettings?.value?.address ?? '',
  });
  
  // Background refresh stale data (no loading shown)
  if (!voiceSettings || this.isStale(voiceSettings)) {
    this.refreshInBackground('voice-settings');
  }
  if (!addressSettings || this.isStale(addressSettings)) {
    this.refreshInBackground('address-settings');
  }
}
```

**Impact on initial load:** +60-70ms (DS reads run in parallel with other IPC calls)

Benchmarked result: ~200ms skeleton → ~260-270ms skeleton. Colleague confirmed this is not very noticeable.

### 6.4 What We Need to Build

#### 6.4.1 Cache Data Structure

```typescript
interface CachedData<T> {
  value: T;
  fetchedAt: number;
}

// Stored in DS as:
// { 'voice-settings': { value: {...}, fetchedAt: 1701234567890 } }
```

**Effort:** Trivial, just a convention.

#### 6.4.2 Staleness Checker

```typescript
const DEFAULT_STALE_TIME = 5 * 60 * 1000; // 5 minutes

function isStale<T>(
  cached: CachedData<T> | null, 
  staleTime: number = DEFAULT_STALE_TIME
): boolean {
  if (!cached) return true;
  return Date.now() - cached.fetchedAt > staleTime;
}
```

**Effort:** ~1 hour

#### 6.4.3 Background Refresh Logic

```typescript
async function refreshInBackground(
  key: string,
  fetcher: () => Promise<any>
): Promise<void> {
  try {
    const fresh = await fetcher();
    await ds.set(key, {
      value: fresh,
      fetchedAt: Date.now(),
    });
    // DS event notifies all subscribers
  } catch (error) {
    // See error handling section
    console.error(`Background refresh failed for ${key}:`, error);
  }
}
```

**Effort:** ~2-3 hours (basic), ~1-2 days (with proper error handling)

#### 6.4.4 Error Handling

What happens when background refresh fails?

**Option A: Ignore errors (simplest)**
```typescript
async refreshInBackground(key, fetcher) {
  try {
    const fresh = await fetcher();
    await ds.set(key, { value: fresh, fetchedAt: Date.now() });
  } catch (error) {
    // Log and ignore - keep showing stale data
    console.error(`Refresh failed for ${key}:`, error);
  }
}
```
- Pro: Simple
- Con: No retry, user sees stale data indefinitely until next manual refresh

**Option B: Track error state**
```typescript
interface CachedData<T> {
  value: T;
  fetchedAt: number;
  lastError?: string;
  lastErrorAt?: number;
}

async refreshInBackground(key, fetcher) {
  try {
    const fresh = await fetcher();
    await ds.set(key, { 
      value: fresh, 
      fetchedAt: Date.now(),
      lastError: undefined,
      lastErrorAt: undefined,
    });
  } catch (error) {
    // Update error state but keep value
    const current = await ds.get(key);
    if (current) {
      await ds.set(key, {
        ...current,
        lastError: error.message,
        lastErrorAt: Date.now(),
      });
    }
  }
}
```
- Pro: UI can show error indicator if desired
- Con: More complex, need to decide what UI does with error state

**Option C: Retry with backoff**
```typescript
async refreshInBackground(key, fetcher, attempt = 1) {
  const MAX_RETRIES = 3;
  const BACKOFF_MS = [1000, 2000, 4000]; // 1s, 2s, 4s
  
  try {
    const fresh = await fetcher();
    await ds.set(key, { value: fresh, fetchedAt: Date.now() });
  } catch (error) {
    if (attempt < MAX_RETRIES) {
      await sleep(BACKOFF_MS[attempt - 1]);
      return this.refreshInBackground(key, fetcher, attempt + 1);
    }
    console.error(`Refresh failed after ${MAX_RETRIES} attempts:`, error);
  }
}
```
- Pro: Handles transient failures
- Con: More complex, need to track in-progress retries

**Effort:** 
- Option A: ~30 minutes
- Option B: ~3-4 hours
- Option C: ~1 day

#### 6.4.5 Request Deduplication (Optional)

Prevent multiple simultaneous refreshes for the same key:

```typescript
const inFlightRefreshes = new Map<string, Promise<void>>();

async function refreshInBackground(key: string, fetcher: () => Promise<any>) {
  // If refresh already in progress, wait for it
  if (inFlightRefreshes.has(key)) {
    return inFlightRefreshes.get(key);
  }
  
  const refreshPromise = (async () => {
    try {
      const fresh = await fetcher();
      await ds.set(key, { value: fresh, fetchedAt: Date.now() });
    } catch (error) {
      console.error(`Refresh failed for ${key}:`, error);
    } finally {
      inFlightRefreshes.delete(key);
    }
  })();
  
  inFlightRefreshes.set(key, refreshPromise);
  return refreshPromise;
}
```

**Effort:** ~3-4 hours

**When is this needed?**
```
Scenario: User navigates quickly

T=0      Alexa Page mounts, data stale, starts refresh
T=100    User taps Voice Setting → Voice Selection
T=200    User taps back (didn't select anything)
T=201    Alexa Page focus, performRefresh() called
T=202    Data still stale (refresh from T=0 not done), starts ANOTHER refresh

Without dedup: Two API calls for same data
With dedup: Second call reuses first promise
```

In React Native with typical navigation, this is uncommon but possible.

#### 6.4.6 Optimistic Updates

```typescript
async selectVoice(newVoice: string) {
  // Get previous value (already in memory via subscription)
  const previousVoice = this.state.voice;
  
  // Optimistic update via DS
  await ds.set('voice-settings', {
    value: { voice: newVoice },
    fetchedAt: Date.now(),
  });
  // DS event updates all subscribers immediately
  
  try {
    await api.saveVoice(newVoice);
    // Success! DS already has correct value
  } catch (error) {
    // Rollback
    await ds.set('voice-settings', {
      value: { voice: previousVoice },
      fetchedAt: Date.now(),
    });
    this.showError('Failed to save voice setting');
  }
}
```

**Note:** No need to read from DS for previous value - it's already in the ViewModel's state from subscription.

**Effort:** Pattern is straightforward, ~30 minutes per mutation

### 6.5 Complete Implementation Effort

| Component | Effort | Priority |
|-----------|--------|----------|
| Cache data structure | 1 hour | P0 |
| Staleness checking | 1 hour | P0 |
| Background refresh (basic) | 3-4 hours | P0 |
| DS event subscription setup | 2-3 hours | P0 |
| Integration with existing load pattern | 4-5 hours | P0 |
| Error handling (Option B: track state) | 3-4 hours | P1 |
| Retry logic | 1 day | P1 |
| Request deduplication | 3-4 hours | P2 |
| Optimistic update pattern | 2-3 hours | P1 |
| Testing | 2-3 days | P0 |

**Total P0:** ~3-4 days
**Total P0+P1:** ~1-1.5 weeks
**Total all:** ~1.5-2 weeks

### 6.6 Pros and Cons

#### Pros

| Advantage | Explanation |
|-----------|-------------|
| ✅ No external dependencies | Uses existing DS infrastructure |
| ✅ Team familiarity | Team already knows DS and EventBus |
| ✅ Persistence is inherent | DS is persistent storage |
| ✅ Cross-screen sync works | Via DS events with full value |
| ✅ Simple mental model | "It's just storage with events" |
| ✅ No bundle size increase | No new libraries |
| ✅ No memory overhead | Data is in DS, not duplicated in memory |
| ✅ Full control | Can customize exactly to our needs |

#### Cons

| Disadvantage | Explanation |
|--------------|-------------|
| ❌ Must build SWR logic | Staleness, refresh, error handling |
| ❌ Edge cases over time | Will discover issues in production |
| ❌ Maintenance burden | Custom code to maintain |
| ❌ No rich status states | Just data or null (unless we build tracking) |
| ❌ DS read latency on mount | 60-70ms per read (but runs in parallel) |
| ❌ Retry logic is custom | Must implement and test ourselves |
| ❌ Deduplication is custom | Must implement if needed |

---

## 7. Option B: TanStack Query Core

### 7.1 What is TanStack Query Core?

TanStack Query (formerly React Query) is the most popular async state management library with **40+ million npm downloads/month**. 

The **core package** (`@tanstack/query-core`) is **framework-agnostic** - it contains all caching logic without React/Vue/Angular dependencies.

Framework-specific packages (React Query, Vue Query, etc.) are thin adapters (~200-400 lines) on top of the core.

We would build an **RxJS adapter** following the same pattern.

### 7.2 What It Provides Out of the Box

| Feature | Description | Our Effort |
|---------|-------------|------------|
| In-memory cache | Map-based storage with query keys | None |
| Sync cache read | `getQueryData()` returns instantly | None |
| Staleness tracking | `staleTime` option, automatic checking | None |
| Background refresh | Automatic when stale, configurable | None |
| Request deduplication | Same query key = same request | None |
| Error handling | Error state per query, keeps stale on error | None |
| Retry logic | Exponential backoff, configurable attempts | None |
| Garbage collection | Auto-cleanup unused queries (`gcTime`) | None |
| Optimistic updates | `onMutate` → update → `onError` rollback | Pattern support |
| Rich status states | `isLoading`, `isFetching`, `isStale`, `error` | None |
| Observer pattern | Multiple subscribers to same query | None |
| Persistence plugin | `@tanstack/query-persist-client-core` | Integration only |

### 7.3 Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                            ViewModel                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  constructor() {                                                     │
│    // SYNC read from in-memory cache (<1ms)                         │
│    const cached = queryClient.getQueryData(['voice-settings']);     │
│    super({                                                           │
│      loading: !cached,                                              │
│      voice: cached?.voice ?? '',                                    │
│    });                                                               │
│  }                                                                   │
│                                                                      │
│  initialize() {                                                      │
│    this.subscription = createQueryObservable({                       │
│      queryKey: ['voice-settings'],                                  │
│      queryFn: () => api.getVoiceSettings(),                         │
│      staleTime: 5 * 60 * 1000,                                      │
│    }).subscribe((result) => {                                        │
│      this.updateState({                                              │
│        voice: result.data?.voice ?? '',                             │
│        loading: result.isLoading,                                   │
│        error: result.error,                                         │
│      });                                                             │
│    });                                                               │
│  }                                                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      RxJS Adapter (~150-200 lines)                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  // Wraps QueryObserver in Observable                               │
│  function createQueryObservable(options) → Observable<QueryResult>  │
│                                                                      │
│  // Sync cache read for constructors                                │
│  function getQueryDataSync(queryKey) → T | undefined                │
│                                                                      │
│  // Direct cache update (for cross-screen sync)                     │
│  function setQueryData(queryKey, data) → void                       │
│                                                                      │
│  // Invalidate cache                                                │
│  function invalidateQueries(queryKey) → Promise<void>               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  @tanstack/query-core (~13KB)                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  QueryClient                                                         │
│    ├─ QueryCache (Map<queryHash, Query>)                            │
│    │    ├─ get(key): returns cached data instantly                  │
│    │    ├─ set(key, data): updates + notifies observers             │
│    │    └─ invalidate(key): marks stale, triggers refetch           │
│    │                                                                 │
│    ├─ MutationCache                                                 │
│    │    └─ Tracks mutations, supports optimistic updates            │
│    │                                                                 │
│    └─ QueryObserver                                                 │
│         ├─ subscribe(callback): get notified of changes             │
│         ├─ getCurrentResult(): sync read                            │
│         └─ Handles SWR, dedup, retry automatically                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Persistence Layer                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  On app start:                                                       │
│    1. Read serialized cache from DS                                 │
│    2. Hydrate QueryClient                                           │
│    3. App ready, queries available instantly                        │
│                                                                      │
│  On cache change:                                                    │
│    1. Dehydrate QueryClient                                         │
│    2. Write serialized cache to DS (async, throttled)               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.4 Cross-Screen Sync Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│ SCENARIO: User changes voice in inner screen, goes back             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  T=0ms      Alexa Page showing "Voice: Default"                     │
│             QueryObserver subscribed to ['voice-settings']          │
│                                                                      │
│  T=100ms    User taps → Voice Selection opens                       │
│                                                                      │
│  T=5000ms   User selects "Voice 3"                                  │
│             Voice Selection calls:                                   │
│               setQueryData(['voice-settings'], { voice: 'Voice 3' })│
│                                                                      │
│  T=5001ms   QueryCache updates (SYNC, <1ms)                         │
│             ALL QueryObservers notified (SYNC)                      │
│             Alexa Page's observer fires                             │
│             Alexa Page BehaviorSubject updated                      │
│             (Persistence writes to DS async in background)          │
│                                                                      │
│  T=5300ms   API save completes                                      │
│                                                                      │
│  T=5500ms   User presses BACK                                       │
│                                                                      │
│  T=5501ms   Alexa Page visible                                      │
│             Already showing "Voice 3" ✓                             │
│             Was updated at T=5001ms!                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

KEY DIFFERENCE FROM DS-ONLY:
- DS-only: Notification is async (~10ms after write)
- TanStack: Notification is SYNC (immediate)

In practice, both are fast enough. But TanStack is deterministically instant.
```

### 7.5 RxJS Adapter Implementation

```typescript
// core/cache/query-adapter.ts

import { 
  QueryClient, 
  QueryObserver,
  MutationObserver,
  QueryObserverResult,
} from '@tanstack/query-core';
import { Observable } from 'rxjs';

// ============================================================
// QUERY CLIENT SINGLETON
// ============================================================

let queryClient: QueryClient | null = null;

export function getQueryClient(): QueryClient {
  if (!queryClient) {
    queryClient = new QueryClient({
      defaultOptions: {
        queries: {
          staleTime: 5 * 60 * 1000,     // 5 minutes
          gcTime: 10 * 60 * 1000,       // GC after 10 minutes unused
          retry: 3,                      // Retry 3 times
          retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 30000),
          refetchOnWindowFocus: false,   // We handle via onFocus
        },
      },
    });
  }
  return queryClient;
}

// ============================================================
// SYNC CACHE ACCESS (for constructors)
// ============================================================

export function getQueryDataSync<T>(queryKey: unknown[]): T | undefined {
  return getQueryClient().getQueryData<T>(queryKey);
}

// ============================================================
// CACHE UPDATES (for cross-screen sync)
// ============================================================

export function setQueryData<T>(queryKey: unknown[], data: T): void {
  getQueryClient().setQueryData(queryKey, data);
}

export function invalidateQueries(queryKey: unknown[]): Promise<void> {
  return getQueryClient().invalidateQueries({ queryKey });
}

// ============================================================
// QUERY OBSERVABLE (main interface)
// ============================================================

export function createQueryObservable<T>(options: {
  queryKey: unknown[];
  queryFn: () => Promise<T>;
  staleTime?: number;
  enabled?: boolean;
}): Observable<QueryObserverResult<T, Error>> {
  return new Observable((subscriber) => {
    const observer = new QueryObserver<T, Error>(
      getQueryClient(),
      options
    );
    
    // Emit current state immediately (sync)
    subscriber.next(observer.getCurrentResult());
    
    // Subscribe to future changes
    const unsubscribe = observer.subscribe((result) => {
      subscriber.next(result);
    });
    
    // Cleanup on unsubscribe
    return () => {
      unsubscribe();
      observer.destroy();
    };
  });
}

// ============================================================
// MUTATION OBSERVABLE (for optimistic updates)
// ============================================================

export function createMutationObservable<TData, TVariables>(options: {
  mutationFn: (variables: TVariables) => Promise<TData>;
  onMutate?: (variables: TVariables) => Promise<unknown> | unknown;
  onSuccess?: (data: TData, variables: TVariables, context: unknown) => void;
  onError?: (error: Error, variables: TVariables, context: unknown) => void;
}) {
  const observer = new MutationObserver(getQueryClient(), options);
  
  return {
    mutate: (variables: TVariables) => observer.mutate(variables),
    mutateAsync: (variables: TVariables) => observer.mutate(variables),
    reset: () => observer.reset(),
  };
}
```

**Lines of code:** ~80-100 (core), ~50-100 (utilities/types)

### 7.6 Persistence Setup

```typescript
// core/cache/query-persistence.ts

import { persistQueryClient } from '@tanstack/query-persist-client-core';
import { getQueryClient } from './query-adapter';
import { ds } from './datastore';

const CACHE_KEY = 'TANSTACK_QUERY_CACHE_V1';

export async function setupQueryPersistence(): Promise<void> {
  const persister = {
    persistClient: async (client: any) => {
      const serialized = JSON.stringify(client);
      await ds.set(CACHE_KEY, serialized);
    },
    restoreClient: async () => {
      const serialized = await ds.get(CACHE_KEY);
      return serialized ? JSON.parse(serialized) : undefined;
    },
    removeClient: async () => {
      await ds.delete(CACHE_KEY);
    },
  };

  await persistQueryClient({
    queryClient: getQueryClient(),
    persister,
    maxAge: 24 * 60 * 60 * 1000, // 24 hours
  });
}

// Called during app startup
async function initializeApp() {
  // Hydrate cache from DS (60-70ms)
  await setupQueryPersistence();
  
  // Now all screens can access cached data instantly
}
```

### 7.7 ViewModel Integration

```typescript
class AlexaPageViewModel extends BaseViewModel<AlexaPageState> {
  private subscription?: Subscription;
  
  constructor() {
    // SYNC cache read (<1ms) - works in constructor!
    const cached = getQueryDataSync<VoiceSettings>(['voice-settings']);
    
    super({
      loading: !cached,
      voice: cached?.voice ?? '',
    });
  }
  
  initialize() {
    // Subscribe to query
    this.subscription = createQueryObservable({
      queryKey: ['voice-settings'],
      queryFn: () => api.getVoiceSettings(),
      staleTime: 5 * 60 * 1000,
    }).subscribe((result) => {
      this.updateState({
        voice: result.data?.voice ?? '',
        loading: result.isLoading,
        // Optionally show error or "refreshing" state
        error: result.error?.message,
        isRefreshing: result.isFetching && !result.isLoading,
      });
    });
  }
  
  performRefresh() {
    // Just invalidate - TanStack handles the rest
    invalidateQueries(['voice-settings']);
  }
  
  dispose() {
    this.subscription?.unsubscribe();
  }
}
```

### 7.8 Optimistic Updates

```typescript
class VoiceSelectionViewModel extends BaseViewModel<VoiceSelectionState> {
  
  selectVoice(newVoice: string) {
    // Get previous value (sync, <1ms)
    const previous = getQueryDataSync<VoiceSettings>(['voice-settings']);
    
    // Optimistic update - notifies all observers instantly
    setQueryData(['voice-settings'], { voice: newVoice });
    
    // API call
    api.saveVoice(newVoice)
      .catch((error) => {
        // Rollback on error
        if (previous) {
          setQueryData(['voice-settings'], previous);
        }
        this.showError('Failed to save voice');
      });
  }
}
```

### 7.9 Implementation Effort

| Component | Effort | Notes |
|-----------|--------|-------|
| RxJS adapter | 1 day | ~150-200 lines |
| Persistence integration | 0.5 days | ~50 lines |
| App initialization | 0.5 days | Setup during startup |
| First ViewModel migration | 1 day | Alexa Page + Voice Selection |
| Remaining ViewModels | 2 days | ~9 more endpoints |
| Testing | 2 days | Integration tests |

**Total: ~1 week**

### 7.10 Bundle Size

| Package | Size | Gzipped |
|---------|------|---------|
| @tanstack/query-core | 13KB | 4KB |
| @tanstack/query-persist-client-core | 3KB | 1KB |
| Our adapter code | 2KB | 1KB |
| **Total** | **18KB** | **6KB** |

Context: Typical React Native app bundle is 2-10MB. This adds 0.1-0.5%.

### 7.11 Memory Usage

| Component | Memory |
|-----------|--------|
| QueryClient instance | ~2KB |
| QueryCache (10 queries × 2KB data) | ~20KB |
| Query metadata (per query) | ~200 bytes |
| **Total** | **~25KB** |

Context: Typical React Native app uses 100-200MB. This is 0.01%.

### 7.12 Pros and Cons

#### Pros

| Advantage | Explanation |
|-----------|-------------|
| ✅ Battle-tested | 40M+ downloads/month, used in production by thousands of apps |
| ✅ All SWR logic built-in | Staleness, refresh, retry, dedup - all handled |
| ✅ Rich status states | `isLoading`, `isFetching`, `isStale`, `error` |
| ✅ Sync cache reads | Works in constructor |
| ✅ Cross-screen sync | Instant via observer pattern |
| ✅ Error handling built-in | Keeps stale data, tracks error state |
| ✅ Retry built-in | Exponential backoff, configurable |
| ✅ Deduplication built-in | Automatic for same query key |
| ✅ Less custom code | ~200 lines adapter vs ~500-1000 lines custom SWR |
| ✅ Persistence plugin | Official, tested, works with our DS |
| ✅ Minimal effort | ~1 week total |
| ✅ Low maintenance | Library maintained by Tanner Linsley + community |

#### Cons

| Disadvantage | Explanation |
|--------------|-------------|
| ❌ External dependency | Adds package to maintain/update |
| ❌ Learning curve | Team needs to learn TanStack concepts |
| ❌ Bundle size | +18KB (6KB gzipped) |
| ❌ Memory overhead | ~25KB in memory |
| ❌ Adapter layer | Custom RxJS adapter to maintain |
| ❌ DS events not used | Doesn't leverage existing DS event system |
| ❌ Cold start hydration | Still need 60-70ms DS read on startup |

---

## 8. Edge Cases Deep Dive

### 8.1 Double Refresh Scenario

**Scenario:** User navigates away and back while refresh is in progress.

```
T=0      Alexa Page mounts
T=1      Cache is stale, starts background refresh
T=100    User taps Settings, then back
T=101    Alexa Page focus, performRefresh() called
T=102    Is refresh still in progress?

DS-ONLY (without dedup):
  T=102  Starts SECOND API call (wasteful)
  T=400  First API completes, writes to DS
  T=500  Second API completes, writes to DS (duplicate)

DS-ONLY (with dedup):
  T=102  Detects refresh in progress, waits
  T=400  First API completes, both resolved

TANSTACK QUERY:
  T=102  Automatic dedup, reuses in-flight request
  T=400  Single API completes, all observers notified
```

**Impact:** Without dedup, waste API calls. Rare but possible.

**DS-Only effort to handle:** +3-4 hours  
**TanStack:** Automatic

### 8.2 Error During Background Refresh

**Scenario:** API fails while refreshing stale data.

```
T=0      Show cached data (stale but valid)
T=1      Start background refresh
T=500    API returns 500 error

DS-ONLY (basic):
  Error logged
  User sees stale data indefinitely
  No indication of error
  Manual refresh needed to try again

DS-ONLY (with error tracking):
  Error stored in cache
  UI can optionally show error indicator
  Next focus will retry

DS-ONLY (with retry):
  First failure: retry after 1s
  Second failure: retry after 2s
  Third failure: give up, log error

TANSTACK QUERY:
  Error tracked in query state (result.error)
  Stale data preserved (result.data still available)
  Automatic retry with exponential backoff
  UI can show: data + "failed to refresh" indicator
```

**DS-Only effort to handle properly:** +1-2 days  
**TanStack:** Built-in

### 8.3 Rapid Screen Transitions

**Scenario:** User taps around quickly.

```
T=0      Screen A mounts, subscribes to query
T=50     Screen A starts fetching
T=100    User navigates to Screen B
T=150    Screen B mounts, subscribes to SAME query
T=200    User navigates to Screen C
T=250    Screen C mounts, subscribes to SAME query
T=400    Original fetch completes

DS-ONLY:
  Each screen subscription needs cleanup
  Fetch was started by Screen A - does it still care?
  When fetch completes, write to DS
  DS event goes to... which screens?
  Screen A might be unmounted
  Need to handle dangling subscriptions

TANSTACK QUERY:
  Query exists in cache, independent of screens
  Multiple observers can attach/detach
  Fetch completes → query state updates
  Only mounted observers receive update
  Garbage collection handles unused queries
```

**DS-Only effort to handle:** Careful subscription management  
**TanStack:** Automatic observer lifecycle

### 8.4 Optimistic Update Rollback

**Scenario:** User changes voice, API fails.

```
T=0      User on Voice Selection, voice = "Default"
T=1      User taps "Voice 3"

DS-ONLY:
  T=2      Get previous value (already in state: "Default")
  T=3      Write to DS: { voice: "Voice 3" }
  T=53     DS write completes, event fires
  T=54     All screens show "Voice 3" (optimistic)
  T=100    API call starts
  T=500    API returns error
  T=501    Write to DS: { voice: "Default" }
  T=551    DS write completes, event fires
  T=552    All screens show "Default" (rollback)
  
  Total time in wrong state: ~500ms (expected)
  Works correctly ✓

TANSTACK QUERY:
  T=2      Get previous value (sync): "Default"
  T=3      setQueryData({ voice: "Voice 3" })
  T=4      All observers notified INSTANTLY
  T=5      All screens show "Voice 3" (optimistic)
  T=100    API call starts
  T=500    API returns error
  T=501    setQueryData({ voice: "Default" })
  T=502    All observers notified INSTANTLY
  T=503    All screens show "Default" (rollback)
  
  Total time in wrong state: ~500ms (expected)
  Works correctly ✓
```

**Difference:** TanStack is slightly faster (~50ms) due to sync updates, but both work correctly.

### 8.5 Background Event Invalidation

**Scenario:** External event requires cache invalidation.

```
T=0      User views settings, cache populated
T=1      App backgrounded, eventually killed
T=X      External event occurs (e.g., device deregistered)
T=X+1    Background task receives notification
T=Y      User reopens app

REQUIREMENT: User must NOT see stale cached data

DS-ONLY:
  Background task: ds.delete('affected-keys')
  On app open: DS read returns null
  Screen shows loading, fetches fresh data ✓

TANSTACK QUERY:
  Background task: 
    // Clear persisted cache
    ds.delete('TANSTACK_QUERY_CACHE_V1')
    // OR: Update a "cache version" in DS
  On app open:
    Hydration finds no cache (or old version)
    Screen shows loading, fetches fresh data ✓
```

**Both work.** DS-Only is slightly more granular (can invalidate specific keys). TanStack invalidates entire cache (but can implement version check).

### 8.6 Cache Size Growth

**Scenario:** Does cache grow unbounded?

```
DS-ONLY:
  We manually manage keys
  ~10 known queries
  Each ~2KB
  Total: ~20KB
  No automatic cleanup
  BUT: We control what gets cached

TANSTACK QUERY:
  gcTime: 10 * 60 * 1000 (10 minutes default)
  Queries not accessed for 10 minutes get garbage collected
  Automatic cleanup ✓
```

**DS-Only:** We need to manage cache size manually (but with ~10 queries, it's not a concern)  
**TanStack:** Automatic garbage collection

---

## 9. Comparison Matrix

### 9.1 Feature Comparison

| Feature | DS-Only | TanStack Query |
|---------|---------|----------------|
| **Cache storage** | DS (persistent) | In-memory + DS backup |
| **Cache read latency** | 60-70ms | <1ms |
| **Use in constructor** | ❌ No (async) | ✅ Yes (sync) |
| **Cross-screen sync** | ✅ Via DS events | ✅ Via observers |
| **Sync notification** | ❌ Async (~10ms) | ✅ Sync (<1ms) |
| **SWR** | ⚠️ Build | ✅ Built-in |
| **Staleness tracking** | ⚠️ Build | ✅ Built-in |
| **Error handling** | ⚠️ Build | ✅ Built-in |
| **Retry logic** | ⚠️ Build | ✅ Built-in |
| **Request deduplication** | ⚠️ Build | ✅ Built-in |
| **Optimistic updates** | ✅ Works | ✅ Works |
| **Rich status states** | ⚠️ Build | ✅ Built-in |
| **Garbage collection** | ❌ Manual | ✅ Automatic |
| **Persistence** | ✅ Inherent | ✅ Via DS |

### 9.2 Effort Comparison

| Component | DS-Only | TanStack |
|-----------|---------|----------|
| Core cache infrastructure | 1-2 days | 0 (built-in) |
| Staleness tracking | 0.5 days | 0 (built-in) |
| Background refresh | 0.5-1 day | 0 (built-in) |
| Error handling | 0.5-1 day | 0 (built-in) |
| Retry logic | 1 day | 0 (built-in) |
| Request deduplication | 0.5 days | 0 (built-in) |
| RxJS adapter | 0 | 1 day |
| Persistence | 0 (inherent) | 0.5 days |
| ViewModel integration | 1-2 days | 1-2 days |
| Testing | 2-3 days | 2 days |
| **Total** | **~1.5-2 weeks** | **~1 week** |

### 9.3 Maintenance Comparison

| Aspect | DS-Only | TanStack |
|--------|---------|----------|
| Code to maintain | ~500-1000 lines | ~200 lines (adapter) |
| Edge cases discovered in prod | Yes, over time | No (library handles) |
| Bug fixes | Our responsibility | Library + our adapter |
| Updates | Our responsibility | Library updates |
| Documentation | Must write | Extensive docs exist |

### 9.4 Risk Comparison

| Risk | DS-Only | TanStack |
|------|---------|----------|
| Undiscovered bugs | Higher | Lower |
| Performance issues | Medium | Low |
| Memory leaks | Medium | Low |
| External dependency | None | @tanstack/query-core |
| Library abandonment | N/A | Low (very active) |
| Breaking changes | N/A | Possible but versioned |

### 9.5 Cold Start Comparison

| Phase | DS-Only | TanStack |
|-------|---------|----------|
| App launch | - | - |
| Hydrate cache | N/A (DS is cache) | Read from DS: 60-70ms |
| First screen mount | Read from DS: 60-70ms | Read from memory: <1ms |
| **Total before data** | **60-70ms** | **60-70ms** |

**Same cold start performance!** TanStack moves the DS read to app initialization instead of per-screen.

---

## 10. Risk Analysis

### 10.1 DS-Only Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Edge case bugs in SWR | High | Medium | Thorough testing, production monitoring |
| Retry logic issues | Medium | Low | Simple retry is straightforward |
| Race conditions | Medium | Medium | Careful async handling |
| Memory leaks from subscriptions | Medium | Medium | Proper cleanup |
| Performance issues | Low | Medium | Benchmarking |

### 10.2 TanStack Query Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Library breaking changes | Low | Medium | Pin version, update carefully |
| Library abandonment | Very Low | High | Well-maintained, active community |
| RxJS adapter bugs | Medium | Medium | Thorough testing |
| Learning curve | Medium | Low | Good documentation |
| Bundle size concerns | Low | Low | 6KB gzipped is minimal |

---

## 11. Recommendation

### 11.1 Decision Framework

| If... | Then... |
|-------|---------|
| Team prefers no external dependencies | DS-Only |
| Team wants to minimize custom code | TanStack Query |
| Caching needs may grow in complexity | TanStack Query |
| Caching needs will stay simple | Either works |
| Development speed is priority | TanStack Query |
| Full control is priority | DS-Only |

### 11.2 Both Options Are Valid

This is a genuine trade-off, not a clear winner:

**DS-Only is the right choice if:**
- Team strongly prefers avoiding dependencies
- Team has bandwidth for 1.5-2 weeks of development
- Team is comfortable maintaining SWR logic long-term
- Caching requirements will stay simple

**TanStack Query is the right choice if:**
- Team wants to minimize custom caching code
- Team values battle-tested solutions
- Caching requirements may grow
- Development speed is important

### 11.3 My Assessment

Given:
- SWR is definitely needed
- ~10 API endpoints is not trivial
- Error handling and retry are valuable
- Team time has opportunity cost

**I lean toward TanStack Query** because:
1. The effort difference (1 week vs 1.5-2 weeks) is real
2. Edge cases in SWR are subtle and will be discovered over time
3. The dependency risk is low (mature, stable library)
4. The bundle size (~6KB gzipped) is negligible
5. The team can focus on product features instead of caching infrastructure

**But DS-Only is completely reasonable** if:
1. The team has strong preference for no dependencies
2. The team is confident in implementing robust SWR
3. The simpler mental model ("it's just storage") is valued

### 11.4 For the Design Review

Present both options honestly. The decision depends on team values:

> "We have two viable approaches. Both meet our P0 requirements. The trade-off is:
>
> **DS-Only:** No dependencies, uses existing infrastructure, ~1.5-2 weeks effort, we own the SWR logic.
>
> **TanStack Query:** External dependency (stable, 40M downloads/month), ~1 week effort, SWR logic is battle-tested.
>
> Both have acceptable performance. Both achieve the goal of eliminating unnecessary loading. The question is: do we want to own caching logic, or use a library?"

---

## 12. Implementation Plan

### 12.1 DS-Only Implementation

**Week 1:**
| Day | Task |
|-----|------|
| 1-2 | Cache data structure, staleness checking, basic background refresh |
| 3 | Error handling + retry logic |
| 4 | DS event subscription integration |
| 5 | First ViewModel migration (Alexa Page) |

**Week 2:**
| Day | Task |
|-----|------|
| 1-2 | Remaining ViewModel migrations |
| 3 | Request deduplication (if needed) |
| 4-5 | Testing + edge case handling |

### 12.2 TanStack Query Implementation

**Week 1:**
| Day | Task |
|-----|------|
| 1 | RxJS adapter implementation |
| 2 | Persistence integration with DS |
| 3 | First ViewModel migration (Alexa Page + Voice Selection) |
| 4 | Remaining ViewModel migrations |
| 5 | Testing + edge case handling |

---

## 13. Open Questions

1. **Team preference on dependencies?**
   - Strong preference for no dependencies → DS-Only
   - Okay with stable dependencies → TanStack Query

2. **Future caching complexity?**
   - Will stay simple → Either works
   - May grow complex → TanStack Query has more features

3. **Available development time?**
   - Tight timeline → TanStack Query (~1 week faster)
   - Flexible → Either works

4. **Maintenance preference?**
   - Prefer maintaining our own code → DS-Only
   - Prefer maintaining minimal adapter → TanStack Query

---

## Summary

| Aspect | DS-Only | TanStack Query |
|--------|---------|----------------|
| **Meets requirements** | ✅ Yes | ✅ Yes |
| **Development effort** | ~1.5-2 weeks | ~1 week |
| **Maintenance burden** | Higher | Lower |
| **External dependency** | None | Yes (stable) |
| **SWR complexity** | We own it | Library handles |
| **Bundle size** | +0KB | +18KB (6KB gzip) |
| **Memory** | In DS only | ~25KB in memory |
| **Cold start** | ~60-70ms | ~60-70ms |
| **Cross-screen sync** | Via DS events | Via observers |
| **Risk** | Edge cases | Dependency |

**Both are valid solutions. The choice depends on team preferences around dependencies vs custom code.**
