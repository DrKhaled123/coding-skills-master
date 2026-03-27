# Advanced Bugs: Async, Concurrency, Memory & Performance

## 🎯 Purpose

Patterns for the hardest category of bugs — those that are intermittent, environment-specific, or only appear under load or over time.

> **VIP EXPERT NOTE — The Heisenbug Principle:**
> The hardest bugs change behavior when you try to observe them. Adding
> console.log changes timing, fixing the environment removes the trigger.
> For these bugs: **instrument first, hypothesize second, change code last.**
> Log with timestamps plus correlation IDs before touching any logic.

---

## 🔷 ADVANCED-01: Race Conditions

### What is a Race Condition?

Two or more operations run concurrently, and the final result depends on which operation completes first. The result is correct sometimes and wrong other times.

> **VIP EXPERT NOTE — Timing is Everything:**
> Race conditions are the most insidious bugs because they appear "random."
> The same code can work 999 times and fail on the 1000th. When debugging,
> always assume timing issues first if the bug is intermittent.

### Detection Signals

```
□ Bug is intermittent — happens sometimes, not always
□ Bug is more frequent under load
□ Bug disappears when you add console.log (changes timing)
□ Multiple concurrent users trigger it but single user doesn't
□ Tests pass individually but fail when run in parallel
□ Bug only appears in production, never in development
□ Adding delays (setTimeout) makes the bug disappear
```

### Pattern: UI Race Condition (Search/Filter)

```typescript
// ❌ BROKEN: Results from old query overwrite results from new query
function SearchComponent() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  useEffect(() => {
    if (query) {
      searchApi(query).then(setResults);
      // User types "react", then quickly types "redux"
      // If "redux" response arrives before "react" response:
      // setResults called with redux results, then react results overwrite them
      // UI shows "react" results for "redux" query — WRONG
    }
  }, [query]);
}

// ✅ FIX: Track which request is current using AbortController
function SearchComponent() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<SearchResult[]>([]);

  useEffect(() => {
    if (!query) {
      setResults([]);
      return;
    }

    const controller = new AbortController();

    searchApi(query, { signal: controller.signal })
      .then(setResults)
      .catch((err) => {
        if (err.name !== 'AbortError') {
          console.error('Search failed:', err);
        }
        // AbortError is expected — previous query was cancelled
      });

    return () => controller.abort(); // cancel previous request on new keypress
  }, [query]);
}
```

> **VIP EXPERT NOTE — AbortController is Your Friend:**
> Modern fetch APIs support AbortController. Always use it for cancellable
> operations. For non-fetch APIs, implement your own cancellation tokens.

### Alternative: Request Deduplication with Request ID

```typescript
// ✅ FIX: Track latest request ID to ignore stale responses
function SearchComponent() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<SearchResult[]>([]);
  const requestIdRef = useRef(0);

  useEffect(() => {
    if (!query) {
      setResults([]);
      return;
    }

    const currentRequestId = ++requestIdRef.current;

    searchApi(query)
      .then((data) => {
        // Only update if this is still the latest request
        if (currentRequestId === requestIdRef.current) {
          setResults(data);
        }
      })
      .catch((err) => {
        if (currentRequestId === requestIdRef.current) {
          console.error('Search failed:', err);
        }
      });

    return () => {
      // Cleanup if needed
    };
  }, [query]);
}
```

### Pattern: Database Race Condition (Check-Then-Act)

```typescript
// ❌ BROKEN: Time-of-check vs time-of-use (TOCTOU) race
async function transferFunds(fromId: string, toId: string, amount: number) {
  const account = await db.accounts.findById(fromId);

  // Another request might deduct funds between check and deduction!
  if (account.balance < amount) throw new Error('Insufficient funds');

  // GAP: balance can change here by another concurrent request
  await db.accounts.update(fromId, { balance: account.balance - amount });
  await db.accounts.update(toId, { balance: existing => existing + amount });
}

// ✅ FIX: Atomic operation with database-level constraint
async function transferFunds(fromId: string, toId: string, amount: number) {
  await db.transaction(async (trx) => {
    // SELECT FOR UPDATE — locks the row for this transaction
    const account = await trx.accounts.findById(fromId, { lock: true });

    if (account.balance < amount) throw new InsufficientFundsError(fromId, amount);

    // Both updates in same atomic transaction — all or nothing
    await trx.accounts.update(fromId, { balance: account.balance - amount });
    await trx.accounts.update(toId, { balance: existing => existing + amount });
  });
}
```

> **VIP EXPERT NOTE — Database Locking Strategy:**
> SELECT FOR UPDATE is pessimistic locking. For high-contention scenarios,
> consider optimistic locking with version numbers. Know your database's
> isolation levels and their implications.

### Pattern: Singleton Initialization Race

```typescript
// ❌ BROKEN: Two callers both see uninitialized, both initialize
let dbConnection: Database | null = null;

async function getDatabase(): Promise<Database> {
  if (!dbConnection) {
    // Two async calls both reach here before either creates the connection
    dbConnection = await createDatabaseConnection(); // called twice!
  }
  return dbConnection;
}

// ✅ FIX: Promise-based singleton — second caller gets same promise
let connectionPromise: Promise<Database> | null = null;

async function getDatabase(): Promise<Database> {
  if (!connectionPromise) {
    connectionPromise = createDatabaseConnection().catch((err) => {
      connectionPromise = null; // reset on failure so next call can retry
      throw err;
    });
  }
  return connectionPromise; // both callers await same promise
}
```

> **VIP EXPERT NOTE — Singleton Pattern Gotcha:**
> The Promise-based singleton is crucial for async initialization. Without it,
> you get the "double initialization" bug that causes connection pool exhaustion.

### Pattern: File System Race Condition

```typescript
// ❌ BROKEN: TOCTOU vulnerability in file operations
async function processFile(filePath: string) {
  const exists = await fs.access(filePath).then(() => true).catch(() => false);
  if (exists) {
    // File could be deleted between check and read!
    const content = await fs.readFile(filePath, 'utf-8');
    return content;
  }
  return null;
}

// ✅ FIX: Try the operation and handle failure
async function processFile(filePath: string): Promise<string | null> {
  try {
    const content = await fs.readFile(filePath, 'utf-8');
    return content;
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === 'ENOENT') {
      return null; // File doesn't exist — expected case
    }
    throw error; // Unexpected error — rethrow
  }
}
```

---

## 🔷 ADVANCED-02: Memory Leaks

### Detection Signals

```
□ Application slows down over time
□ Memory usage grows steadily without plateau
□ Performance degrades after extended use
□ Crash only happens after N hours of runtime
□ Heap snapshot shows growing objects of same type
□ Garbage collection frequency increases over time
□ "JavaScript heap out of memory" errors in production
```

> **VIP EXPERT NOTE — Memory Leak Signature:**
> The telltale sign is memory that grows without bound. In a healthy app,
> memory should plateau after warm-up. If it keeps climbing, you have a leak.

### Pattern: Event Listener Leak

```typescript
// ❌ BROKEN: Event listener added but never removed
function ModalComponent() {
  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') closeModal();
    };
    document.addEventListener('keydown', handleKeyDown);
    // Component unmounts → listener remains → memory leak + multiple handlers
  }, []);
}

// ✅ FIX: Return cleanup function from useEffect
function ModalComponent() {
  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') closeModal();
    };

    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown); // cleanup
  }, [closeModal]);
}

// Node.js / General pattern:
class EventDrivenService {
  private readonly handlers = new Map<string, EventHandler>();

  subscribe(event: string, handler: EventHandler): () => void {
    emitter.on(event, handler);
    this.handlers.set(event, handler);

    // Return unsubscribe function
    return () => {
      emitter.off(event, handler);
      this.handlers.delete(event);
    };
  }

  destroy(): void {
    // Clean up all handlers
    for (const [event, handler] of this.handlers) {
      emitter.off(event, handler);
    }
    this.handlers.clear();
  }
}
```

### Pattern: Timer/Interval Leak

```typescript
// ❌ BROKEN: Timer continues after component unmounts
function LiveTimer() {
  const [time, setTime] = useState(new Date());

  useEffect(() => {
    setInterval(() => setTime(new Date()), 1000);
    // Timer runs forever — even after component unmounts
    // Calling setState after unmount → React warning + potential leak
  }, []);
}

// ✅ FIX
function LiveTimer() {
  const [time, setTime] = useState(new Date());

  useEffect(() => {
    const timerId = setInterval(() => setTime(new Date()), 1_000);
    return () => clearInterval(timerId); // clear on unmount
  }, []);

  return <span>{time.toLocaleTimeString()}</span>;
}

// ✅ FIX: Using ref for mutable timer ID
function PollingComponent() {
  const [data, setData] = useState(null);
  const intervalRef = useRef<NodeJS.Timeout | null>(null);

  const startPolling = useCallback(() => {
    if (intervalRef.current) return; // Prevent multiple intervals
    intervalRef.current = setInterval(async () => {
      const newData = await fetchData();
      setData(newData);
    }, 5000);
  }, []);

  const stopPolling = useCallback(() => {
    if (intervalRef.current) {
      clearInterval(intervalRef.current);
      intervalRef.current = null;
    }
  }, []);

  useEffect(() => {
    startPolling();
    return stopPolling; // Cleanup on unmount
  }, [startPolling, stopPolling]);
}
```

### Pattern: Closure Holding Large Object

```typescript
// ❌ BROKEN: Closure retains entire largeData in memory
function processLargeDataset(largeData: LargeObject[]) {
  const count = largeData.length;

  // This closure captures the entire largeData array!
  return {
    getCount: () => count,            // OK — only uses count
    processItem: (i: number) => largeData[i], // keeps largeData alive
    // If processItem is stored anywhere (cache, global), largeData never GC'd
  };
}

// ✅ FIX: Extract only what you need, don't close over large objects
function processLargeDataset(largeData: LargeObject[]) {
  const count = largeData.length;
  // Process immediately, don't store reference
  const processedItems = largeData.map(item => extractNeededFields(item));
  // largeData can now be GC'd

  return {
    getCount: () => count,
    getItem: (i: number) => processedItems[i], // only holds processed data
  };
}

// ✅ FIX: WeakRef for cache that shouldn't prevent GC
class WeakValueCache<K, V extends object> {
  private cache = new Map<K, WeakRef<V>>();

  set(key: K, value: V): void {
    this.cache.set(key, new WeakRef(value));
  }

  get(key: K): V | undefined {
    const ref = this.cache.get(key);
    return ref?.deref();
  }

  // Clean up dead references periodically
  cleanup(): void {
    for (const [key, ref] of this.cache) {
      if (ref.deref() === undefined) {
        this.cache.delete(key);
      }
    }
  }
}
```

> **VIP EXPERT NOTE — The Closure Trap:**
> JavaScript closures capture entire scope chains, not just used variables.
> A function that only uses one property of an object still keeps the whole
> object alive. Extract what you need early to enable garbage collection.

---

## 🔷 ADVANCED-03: Stale Closures

> **VIP EXPERT NOTE — The Closure Time Machine:**
> Closures capture values at creation time, not at execution time. This is
> the root cause of many "why is my state old?" bugs in React.

### Pattern: Stale State in Async Callback

```typescript
// ❌ BROKEN: Value in callback is stale
function PollComponent() {
  const [data, setData] = useState<DataItem[]>([]);

  useEffect(() => {
    const pollInterval = setInterval(async () => {
      const newData = await fetchLatestData();
      // STALE: data here always refers to the initial [] array
      // because the closure captured data at component mount
      const combined = [...data, ...newData]; // data is always []!
      setData(combined);
    }, 5000);

    return () => clearInterval(pollInterval);
  }, []); // empty deps — captures stale data
}

// ✅ FIX A: Use functional update (doesn't close over state)
useEffect(() => {
  const pollInterval = setInterval(async () => {
    const newData = await fetchLatestData();
    setData(prev => [...prev, ...newData]); // prev is always current
  }, 5000);
  return () => clearInterval(pollInterval);
}, []);

// ✅ FIX B: Use ref to track current value
function PollComponent() {
  const [data, setData] = useState<DataItem[]>([]);
  const dataRef = useRef(data);

  useEffect(() => {
    dataRef.current = data; // keep ref in sync
  }, [data]);

  useEffect(() => {
    const pollInterval = setInterval(async () => {
      const newData = await fetchLatestData();
      setData([...dataRef.current, ...newData]); // ref is always current
    }, 5000);
    return () => clearInterval(pollInterval);
  }, []);
}
```

### Pattern: Stale Callback in Event Handlers

```typescript
// ❌ BROKEN: Callback uses stale values
function Counter() {
  const [count, setCount] = useState(0);
  const [step, setStep] = useState(1);

  useEffect(() => {
    const handleClick = () => {
      setCount(count + step); // step is always initial value!
    };
    document.addEventListener('click', handleClick);
    return () => document.removeEventListener('click', handleClick);
  }, []); // Missing deps — stale closure

  return <div>Count: {count}</div>;
}

// ✅ FIX: Include all dependencies
useEffect(() => {
  const handleClick = () => {
    setCount(count + step);
  };
  document.addEventListener('click', handleClick);
  return () => document.removeEventListener('click', handleClick);
}, [count, step]); // All deps included

// ✅ FIX: Use functional update to avoid dependency
useEffect(() => {
  const handleClick = () => {
    setCount(c => c + step); // Only need step in deps
  };
  document.addEventListener('click', handleClick);
  return () => document.removeEventListener('click', handleClick);
}, [step]);
```

---

## 🔷 ADVANCED-04: Concurrency Bugs

### Pattern: Concurrent Mutation of Shared State

```typescript
// ❌ BROKEN: Concurrent updates lose data (read-modify-write race)
let viewCount = 0;

async function incrementViewCount(articleId: string): Promise<number> {
  viewCount = viewCount + 1; // not atomic — lost updates under concurrency
  await saveViewCount(articleId, viewCount);
  return viewCount;
}
// 100 concurrent requests: many reads same value, all increment to same value
// Result: viewCount = 1 instead of 100

// ✅ FIX A: Database atomic increment
async function incrementViewCount(articleId: string): Promise<number> {
  const result = await db.query<{ new_count: number }>(
    'UPDATE articles SET view_count = view_count + 1 WHERE id = $1 RETURNING view_count AS new_count',
    [articleId]
  );
  return result.rows[0].new_count;
}

// ✅ FIX B: Mutex for in-memory operations
class AsyncMutex {
  private queue: Array<() => void> = [];
  private locked = false;

  async acquire(): Promise<() => void> {
    return new Promise((resolve) => {
      const tryAcquire = () => {
        if (!this.locked) {
          this.locked = true;
          resolve(() => {
            this.locked = false;
            if (this.queue.length > 0) this.queue.shift()!();
          });
        } else {
          this.queue.push(tryAcquire);
        }
      };
      tryAcquire();
    });
  }
}

const counterMutex = new AsyncMutex();
let viewCount = 0;

async function incrementViewCount(): Promise<number> {
  const release = await counterMutex.acquire();
  try {
    viewCount++;
    return viewCount;
  } finally {
    release();
  }
}
```

> **VIP EXPERT NOTE — Atomic Operations:**
> Database-level atomic operations are always preferred over application-level
> locking. They're faster, more reliable, and handle distributed scenarios.

### Pattern: Deadlock

```typescript
// ❌ BROKEN: Deadlock when two operations try to acquire locks in opposite order
async function transferA_to_B() {
  const lockA = await acquireLock('account-A');
  const lockB = await acquireLock('account-B'); // waits for B
  // ...
}

async function transferB_to_A() {
  const lockB = await acquireLock('account-B');
  const lockA = await acquireLock('account-A'); // waits for A — DEADLOCK
  // ...
}

// ✅ FIX: Always acquire locks in consistent order (alphabetical, numeric, etc.)
async function transfer(fromId: string, toId: string) {
  // Sort IDs so locks are always acquired in same order
  const [firstId, secondId] = [fromId, toId].sort();
  const firstLock = await acquireLock(firstId);
  const secondLock = await acquireLock(secondId);

  try {
    // ... transfer logic
  } finally {
    secondLock.release();
    firstLock.release();
  }
}
```

> **VIP EXPERT NOTE — Deadlock Prevention:**
> Total ordering of resources is the only reliable deadlock prevention.
> Timeouts help detect deadlocks but don't prevent them.

---

## 🔷 ADVANCED-05: Performance Bugs

### Pattern: Blocking the Event Loop (Node.js)

```typescript
// ❌ BROKEN: Synchronous CPU work blocks all other requests
app.get('/api/report', (req, res) => {
  const report = generateHeavyReport(largeDataset); // blocks for 5 seconds
  // All other requests queued — server appears down
  res.json(report);
});

// ✅ FIX A: Move to worker thread
import { Worker, isMainThread, parentPort, workerData } from 'worker_threads';

app.get('/api/report', async (req, res) => {
  const report = await runInWorker('./reportWorker.js', { dataset: largeDataset });
  res.json(report);
});

async function runInWorker<T>(scriptPath: string, data: unknown): Promise<T> {
  return new Promise((resolve, reject) => {
    const worker = new Worker(scriptPath, { workerData: data });
    worker.on('message', resolve);
    worker.on('error', reject);
    worker.on('exit', (code) => {
      if (code !== 0) reject(new Error(`Worker exited with code ${code}`));
    });
  });
}

// ✅ FIX B: Chunk work across event loop ticks
async function generateHeavyReportAsync(chunks: DataChunk[]): Promise<Report> {
  const results = [];
  for (const chunk of chunks) {
    results.push(processChunk(chunk));
    await new Promise(resolve => setImmediate(resolve)); // yield to event loop
  }
  return combineResults(results);
}
```

> **VIP EXPERT NOTE — Event Loop Sanity:**
> Node.js is single-threaded. Any CPU-intensive work blocks ALL requests.
> Use worker_threads for CPU-bound tasks. The event loop is for I/O, not computation.

### Pattern: Cascading API Failures

```typescript
// ❌ BROKEN: One slow/failed service blocks everything
async function loadDashboard(userId: string) {
  const user = await fetchUser(userId);           // if this is slow (2s)...
  const orders = await fetchOrders(userId);       // this waits
  const recommendations = await fetchRecs(userId); // this waits
  // Total: up to 6 seconds — should be 2s (longest single call)
}

// ✅ FIX: Parallel fetch with individual error handling
async function loadDashboard(userId: string): Promise<DashboardData> {
  const [userResult, ordersResult, recsResult] = await Promise.allSettled([
    fetchUser(userId).then(data => ({ type: 'user' as const, data })),
    fetchOrders(userId).then(data => ({ type: 'orders' as const, data })),
    fetchRecs(userId).then(data => ({ type: 'recs' as const, data })),
  ]);

  // Each service failure is isolated — dashboard still works partially
  return {
    user: userResult.status === 'fulfilled' ? userResult.value.data : null,
    orders: ordersResult.status === 'fulfilled' ? ordersResult.value.data : [],
    recommendations: recsResult.status === 'fulfilled' ? recsResult.value.data : [],
  };
}
```

### Pattern: Retry with Exponential Backoff

```typescript
// ❌ WRONG: Retry immediately — hammers failing service
async function fetchWithRetry<T>(url: string): Promise<T> {
  for (let i = 0; i < 3; i++) {
    try {
      return await fetch(url).then(r => r.json());
    } catch {
      // retry immediately — makes outage worse
    }
  }
  throw new Error('Max retries exceeded');
}

// ✅ CORRECT: Exponential backoff with jitter
async function fetchWithRetry<T>(
  url: string,
  options: { maxRetries?: number; baseDelayMs?: number } = {}
): Promise<T> {
  const { maxRetries = 3, baseDelayMs = 1000 } = options;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url);
      if (!response.ok && response.status >= 500 && attempt < maxRetries) {
        throw new Error(`HTTP ${response.status}`); // retry on server errors
      }
      return response.json() as Promise<T>;
    } catch (error) {
      if (attempt === maxRetries) throw error;

      const delay = baseDelayMs * Math.pow(2, attempt) + Math.random() * 1000; // jitter
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
  throw new Error('unreachable');
}
```

> **VIP EXPERT NOTE — Backoff Strategy:**
> Exponential backoff prevents thundering herd. Jitter prevents synchronized retries.
> Never retry immediately — it amplifies outages.

---

## 🔷 ADVANCED-06: Circuit Breaker Pattern

> **VIP EXPERT NOTE — Circuit Breaker Principle:**
> When an external service is down, continuing to call it makes everything
> worse (thread exhaustion, cascading timeouts). The circuit breaker stops
> calling after N failures and retries after a cooldown.

```typescript
class CircuitBreaker {
  private failures = 0;
  private lastFailureTime = 0;
  private state: 'closed' | 'open' | 'half-open' = 'closed';

  constructor(
    private readonly maxFailures: number = 5,
    private readonly cooldownMs: number = 30_000,
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailureTime > this.cooldownMs) {
        this.state = 'half-open'; // Allow one test request
      } else {
        throw new CircuitOpenError('Circuit breaker is open — service unavailable');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    this.failures = 0;
    this.state = 'closed';
  }

  private onFailure(): void {
    this.failures++;
    this.lastFailureTime = Date.now();
    if (this.failures >= this.maxFailures) {
      this.state = 'open';
    }
  }
}

// Usage:
const paymentCircuit = new CircuitBreaker(5, 30_000);

async function processPayment(data: PaymentInput): Promise<PaymentResult> {
  return paymentCircuit.execute(() => paymentGateway.charge(data));
}
```

### Advanced Circuit Breaker with Metrics

```typescript
interface CircuitBreakerMetrics {
  state: 'closed' | 'open' | 'half-open';
  failureCount: number;
  successCount: number;
  lastFailureTime: number;
  rejectionCount: number;
}

class AdvancedCircuitBreaker {
  private failures = 0;
  private successes = 0;
  private rejections = 0;
  private lastFailureTime = 0;
  private state: 'closed' | 'open' | 'half-open' = 'closed';

  constructor(
    private readonly config: {
      failureThreshold: number;
      successThreshold: number;
      timeoutMs: number;
      halfOpenMaxCalls: number;
    }
  ) {}

  getMetrics(): CircuitBreakerMetrics {
    return {
      state: this.state,
      failureCount: this.failures,
      successCount: this.successes,
      lastFailureTime: this.lastFailureTime,
      rejectionCount: this.rejections,
    };
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailureTime > this.config.timeoutMs) {
        this.state = 'half-open';
        this.failures = 0;
        this.successes = 0;
      } else {
        this.rejections++;
        throw new CircuitOpenError('Circuit is open');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    this.successes++;
    if (this.state === 'half-open' && this.successes >= this.config.successThreshold) {
      this.state = 'closed';
      this.failures = 0;
      this.successes = 0;
    }
  }

  private onFailure(): void {
    this.failures++;
    this.lastFailureTime = Date.now();
    if (this.failures >= this.config.failureThreshold) {
      this.state = 'open';
    }
  }
}
```

---

## 🔷 ADVANCED-07: Graceful Shutdown

```typescript
// ❌ BROKEN: Hard shutdown — in-flight requests are dropped, DB connections leak
process.on('SIGTERM', () => process.exit(0));

// ✅ CORRECT: Graceful shutdown — finish in-flight work, close connections
function createGracefulShutdown(server: http.Server, resources: Closeable[]) {
  let isShuttingDown = false;

  async function shutdown(signal: string): Promise<void> {
    if (isShuttingDown) return;
    isShuttingDown = true;
    console.log(`\n[${signal}] Graceful shutdown started...`);

    // 1. Stop accepting new requests
    server.close(() => console.log('HTTP server closed'));

    // 2. Wait for in-flight requests (with timeout)
    const timeout = setTimeout(() => {
      console.error('Shutdown timeout — forcing exit');
      process.exit(1);
    }, 30_000);

    // 3. Close resources (DB pools, Redis, message queues)
    for (const resource of resources) {
      try {
        await resource.close();
        console.log(`Closed: ${resource.constructor.name}`);
      } catch (error) {
        console.error(`Error closing ${resource.constructor.name}:`, error);
      }
    }

    clearTimeout(timeout);
    console.log('Shutdown complete');
    process.exit(0);
  }

  process.on('SIGTERM', () => shutdown('SIGTERM'));
  process.on('SIGINT', () => shutdown('SIGINT'));
}
```

> **VIP EXPERT NOTE — Graceful Shutdown Importance:**
> Kubernetes and other orchestrators send SIGTERM before killing pods.
> Without graceful shutdown: in-flight requests fail, data is lost,
> and you get 502 errors. Always implement proper shutdown handlers.

---

## 🔷 ADVANCED DEBUGGING TECHNIQUES

### Heap Snapshot Analysis (Chrome/Node.js)

```javascript
// In Chrome DevTools or Node.js:
// 1. Take heap snapshot
// 2. Compare snapshots over time
// 3. Look for growing object counts
// 4. Find retaining paths

// Node.js programmatic snapshot
const { writeHeapSnapshot } = require('v8');
writeHeapSnapshot('/tmp/heapdump-' + Date.now() + '.heapsnapshot');

// Analyze memory growth programmatically
class MemoryProfiler {
  private snapshots: Array<{ timestamp: number; usage: number }> = [];

  record(): void {
    const usage = process.memoryUsage();
    this.snapshots.push({
      timestamp: Date.now(),
      usage: usage.heapUsed,
    });
  }

  detectLeak(thresholdPercent: number = 50): boolean {
    if (this.snapshots.length < 10) return false;
    
    const first = this.snapshots[0].usage;
    const last = this.snapshots[this.snapshots.length - 1].usage;
    const growthPercent = ((last - first) / first) * 100;
    
    return growthPercent > thresholdPercent;
  }
}
```

### Async Stack Traces

```javascript
// Enable async stack traces in Node.js
node --async-stack-traces script.js

// In Chrome DevTools:
// Settings -> Enable async stack traces

// Long stack traces for debugging
const { AsyncHook } = require('async_hooks');

class AsyncTracer {
  private contexts = new Map<number, string>();

  constructor() {
    const hook = new AsyncHook({
      init: (asyncId, type, triggerAsyncId) => {
        this.contexts.set(asyncId, `${type}#${asyncId}`);
      },
      destroy: (asyncId) => {
        this.contexts.delete(asyncId);
      },
    });
    hook.enable();
  }

  getContext(asyncId: number): string | undefined {
    return this.contexts.get(asyncId);
  }
}
```

### Logging for Race Conditions

```typescript
// Use correlation IDs to trace requests
function logWithContext(context: string, ...args: unknown[]) {
  const correlationId = getCorrelationId();
  const timestamp = new Date().toISOString();
  console.log(`[${timestamp}] [${correlationId}] [${context}]`, ...args);
}

// Log at key points
async function processRequest(request: Request) {
  logWithContext('processRequest', 'start', { id: request.id });
  
  const step1 = await step1(request);
  logWithContext('processRequest', 'step1 complete', { result: step1.id });
  
  const step2 = await step2(step1);
  logWithContext('processRequest', 'step2 complete', { result: step2.id });
  
  return step2;
}
```

---

## 🔷 VIP EXPERT NOTE: STRUCTURED LOGGING & OBSERVABILITY

> The difference between "we found the bug in 5 minutes" and "we spent 3 days" is
> almost always the quality of logging. Structured logs are searchable; console.log is not.

```typescript
// ❌ WRONG: Unstructured logging
console.log('User created:', userId);
console.log('Error!', error);

// ✅ CORRECT: Structured logging with context
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: process.env.NODE_ENV === 'development'
    ? { target: 'pino-pretty' }   // Human readable in dev
    : undefined,                    // JSON in production
});

// Create child logger with request context
function createRequestLogger(req: Request) {
  return logger.child({
    requestId: req.headers['x-request-id'] ?? crypto.randomUUID(),
    method: req.method,
    path: req.path,
    userId: req.user?.id,
  });
}

// Usage in route handler:
app.post('/api/orders', async (req, res) => {
  const log = createRequestLogger(req);
  log.info({ input: req.body }, 'Creating order');

  try {
    const order = await orderService.create(req.body);
    log.info({ orderId: order.id }, 'Order created successfully');
    return res.status(201).json(order);
  } catch (error) {
    if (error instanceof ValidationError) {
      log.warn({ fields: error.fields }, 'Order validation failed');
      return res.status(422).json({ error: error.fields });
    }
    log.error({ error }, 'Unexpected error creating order');
    return res.status(500).json({ error: 'Internal server error' });
  }
});

// The structured output looks like:
// {"level":30,"time":1711234567890,"requestId":"abc-123","userId":"usr_456",
//  "method":"POST","path":"/api/orders","orderId":"ord_789","msg":"Order created successfully"}
// This is searchable, filterable, and alertable in production logging systems.
```

### Key Logging Rules:

```
✅ DO log: request IDs, user IDs, operation outcomes, timing, error details
✅ DO use: structured JSON format, correlation IDs, log levels
✅ DO include: enough context to reproduce the issue without accessing the codebase

❌ NEVER log: passwords, API keys, tokens, credit card numbers, PII
❌ NEVER use: console.log in production (use a proper logger)
❌ NEVER: log-and-continue without handling the error
```

### Distributed Tracing

```typescript
// OpenTelemetry-style tracing
interface Span {
  traceId: string;
  spanId: string;
  parentSpanId?: string;
  operation: string;
  startTime: number;
  endTime?: number;
  tags: Record<string, unknown>;
}

class Tracer {
  private spans: Span[] = [];

  startSpan(operation: string, parentSpan?: Span): Span {
    const span: Span = {
      traceId: parentSpan?.traceId ?? generateTraceId(),
      spanId: generateSpanId(),
      parentSpanId: parentSpan?.spanId,
      operation,
      startTime: Date.now(),
      tags: {},
    };
    this.spans.push(span);
    return span;
  }

  finishSpan(span: Span): void {
    span.endTime = Date.now();
    // Export to tracing backend (Jaeger, Zipkin, etc.)
  }

  withSpan<T>(operation: string, fn: (span: Span) => Promise<T>): Promise<T> {
    const span = this.startSpan(operation);
    return fn(span).finally(() => this.finishSpan(span));
  }
}
```

---

## 🔷 WHEN TO ESCALATE

If you encounter these scenarios, escalate to senior engineers or use specialized debugging:

- Distributed system deadlocks across services
- Database deadlock analysis with row-level locks
- Low-level memory corruption (C/C++/Rust)
- Kernel-level issues
- Production incidents requiring immediate resolution
- Security vulnerabilities discovered during debugging
- Race conditions in critical financial transactions

> **VIP EXPERT NOTE — Know Your Limits:**
> Advanced bugs can consume days of unproductive time. Set a time limit
> (e.g., 2 hours) for solo debugging. If you haven't made progress, escalate.
> The cost of delay often exceeds the cost of involving help.

---

*This module covers advanced debugging scenarios. Use systematic debugging methodology and escalate when needed.*
