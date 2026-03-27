# Fix Patterns by Bug Category

## 🎯 Purpose

Proven fix patterns for every major bug category. For each pattern: see the broken code, understand why it's broken, apply the fix.

**CRITICAL**: Every fix MUST include a test that would have caught the bug.

> **VIP EXPERT NOTE — Fix Selection Priority:**
> When multiple fixes are possible, choose by this priority:
> 1. **Eliminate the possibility** (make the bug impossible by design)
> 2. **Fail loudly** (throw with clear error if invalid state occurs)
> 3. **Degrade gracefully** (fallback to safe default with logging)
> 4. **Guard at boundary** (validate input at entry point)
> Never: silently swallow, log-and-continue, or return undefined.

---

## 🔷 FIX-01: Null/Undefined Safety

### Pattern: Defensive Access Chain

```typescript
// ❌ BROKEN: Crashes when any link in chain is null
const city = user.address.city.toUpperCase();

// ✅ FIX A: Optional chaining + nullish coalescing (modern JS)
const city = user?.address?.city?.toUpperCase() ?? 'UNKNOWN';

// ✅ FIX B: Guard clauses (when you need to branch on missing data)
function getUserCity(user: User | null): string {
  if (!user) return 'No user';
  if (!user.address) return 'No address';
  if (!user.address.city) return 'No city';
  return user.address.city.toUpperCase();
}

// ✅ FIX C: Default objects at creation time (best for deep nesting)
interface User {
  address: {
    city: string;
    country: string;
  };
}
function createUser(data: Partial<User>): User {
  return {
    address: {
      city: data.address?.city ?? '',
      country: data.address?.country ?? '',
    },
  };
}
// Now user.address.city is ALWAYS a string, never null/undefined
```

### Pattern: Safe Array Access

```typescript
// ❌ BROKEN: Crashes when array is empty or index is out of bounds
const first = items[0].id;
const last = items[items.length - 1].name;

// ✅ FIX
const first = items.at(0); // returns undefined instead of crashing
const last = items.at(-1);  // negative index from end

// With explicit handling:
const first = items.length > 0 ? items[0] : null;
if (!first) return handleEmpty();

// Helper function for safe access:
function safeGet<T>(arr: T[], index: number): T | undefined {
  return index >= 0 && index < arr.length ? arr[index] : undefined;
}

// Safe array operations with type narrowing:
function safeFirst<T>(arr: T[] | null | undefined): T | undefined {
  return arr?.at(0);
}

function safeLast<T>(arr: T[] | null | undefined): T | undefined {
  return arr?.at(-1);
}

// Type-safe find with predicate
function safeFind<T>(
  arr: T[] | null | undefined,
  predicate: (item: T) => boolean
): T | undefined {
  return arr?.find(predicate);
}
```

### Pattern: Null Object Pattern

```typescript
// ❌ BROKEN: Every caller must check for null
interface User {
  id: string;
  name: string;
  preferences: UserPreferences | null;
}

function getTheme(user: User): string {
  return user.preferences?.theme ?? 'light'; // must handle null
}

// ✅ FIX: Null Object Pattern — default instance that does nothing
const DEFAULT_PREFERENCES: UserPreferences = {
  theme: 'light',
  language: 'en',
  notifications: true,
};

function createUser(data: Partial<User>): User {
  return {
    id: data.id ?? crypto.randomUUID(),
    name: data.name ?? 'Anonymous',
    preferences: data.preferences ?? DEFAULT_PREFERENCES,
  };
}

// Now callers don't need null checks:
function getTheme(user: User): string {
  return user.preferences.theme; // always safe
}
```

> **VIP EXPERT NOTE:** Null/undefined errors account for ~70% of JavaScript runtime crashes. 
> Design your types so null is impossible (use empty strings, default objects, discriminated unions).
> Runtime null checks are a safety net — type safety at compile time is the goal.

---

## 🔷 FIX-02: Async/Promise Errors

### Pattern: Missing Await

```typescript
// ❌ BROKEN: Returns Promise object instead of value
function getUserName(id: string) {
  const user = fetchUser(id); // missing await — user is a Promise!
  return user.name;           // undefined — Promise has no .name property
}

// ✅ FIX: Always await async operations
async function getUserName(id: string): Promise<string> {
  const user = await fetchUser(id);
  return user.name;
}

// DETECTION: If you see this pattern, it's almost always missing await:
// someVariable.then(...)  where someVariable looks like it should be a value
// typeof result === 'object' && result.constructor.name === 'Promise'

// ESLint rule to catch this:
// '@typescript-eslint/no-floating-promises': 'error'
```

### Pattern: Unhandled Promise Rejection

```typescript
// ❌ BROKEN: Silent failure — rejection is lost
async function loadDashboard() {
  fetchUserData().then(data => setUser(data));   // rejection not caught
  fetchMetrics().then(data => setMetrics(data)); // rejection not caught
}

// ✅ FIX A: Use Promise.allSettled for independent operations
async function loadDashboard() {
  const [userResult, metricsResult] = await Promise.allSettled([
    fetchUserData(),
    fetchMetrics(),
  ]);

  if (userResult.status === 'fulfilled') setUser(userResult.value);
  else console.error('Failed to load user:', userResult.reason);

  if (metricsResult.status === 'fulfilled') setMetrics(metricsResult.value);
  else console.error('Failed to load metrics:', metricsResult.reason);
}

// ✅ FIX B: Use Promise.all when ALL must succeed
async function loadDashboard() {
  try {
    const [user, metrics] = await Promise.all([fetchUserData(), fetchMetrics()]);
    setUser(user);
    setMetrics(metrics);
  } catch (error) {
    setError(error instanceof Error ? error : new Error(String(error)));
  }
}

// ✅ FIX C: Top-level await with proper error boundaries
async function initializeApp() {
  try {
    const config = await loadConfig();
    const user = await authenticateUser(config);
    return { success: true, user };
  } catch (error) {
    return { 
      success: false, 
      error: error instanceof Error ? error.message : 'Unknown error' 
    };
  }
}
```

### Pattern: Race Condition in State Updates

```typescript
// ❌ BROKEN: If fetchUser is called twice quickly, second response might 
// arrive before first, resulting in stale data displayed
function UserComponent({ userId }: { userId: string }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetchUser(userId).then(setUser); // no cleanup!
  }, [userId]);
}

// ✅ FIX: Cancellation flag pattern
function UserComponent({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    let cancelled = false; // cancellation flag

    fetchUser(userId).then((data) => {
      if (!cancelled) setUser(data); // only update if still mounted
    });

    return () => { cancelled = true; }; // cleanup on unmount or userId change
  }, [userId]);
}

// ✅ FIX (Advanced): AbortController for true cancellation
function UserComponent({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    fetchUser(userId, { signal: controller.signal })
      .then(setUser)
      .catch((err) => {
        if (err.name !== 'AbortError') throw err;
      });

    return () => controller.abort();
  }, [userId]);
}

// ✅ FIX (Modern): React 19+ built-in cancellation
// React 19 automatically handles cleanup for async operations in useEffect
```

### Pattern: Promise Chain Error Handling

```typescript
// ❌ BROKEN: Errors in .then() are not caught by preceding .catch()
fetchUser()
  .then(user => {
    return fetchOrders(user.id); // if this throws, next .catch catches it
  })
  .then(orders => {
    processOrders(orders); // if this throws, it's caught
  })
  .catch(error => {
    // catches errors from fetchUser, fetchOrders, and processOrders
    // can't distinguish where error originated
  });

// ✅ FIX: Separate error handling per operation with specific messages
async function loadUserData() {
  try {
    const user = await fetchUser();
    
    let orders: Order[];
    try {
      orders = await fetchOrders(user.id);
    } catch (error) {
      throw new Error(`Failed to fetch orders for user ${user.id}: ${error}`);
    }
    
    try {
      return processOrders(orders);
    } catch (error) {
      throw new Error(`Failed to process ${orders.length} orders: ${error}`);
    }
  } catch (error) {
    logger.error('loadUserData failed', { error });
    throw error;
  }
}
```

> **VIP EXPERT NOTE:** Async errors are particularly dangerous because they often surface 
> far from their source. Always await promises immediately, handle rejections at the 
> nearest appropriate boundary, and never let a promise rejection go unhandled.

---

## 🔷 FIX-03: React-Specific Bugs

### Pattern: Stale Closure in useEffect

```typescript
// ❌ BROKEN: count is captured at initial render, never updates
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => {
      console.log(count); // always logs 0 — stale closure!
      setCount(count + 1); // always sets to 1 (0 + 1)
    }, 1000);
    return () => clearInterval(timer);
  }, []); // empty deps — closure captures initial count
}

// ✅ FIX A: Add count to deps (re-runs effect when count changes)
useEffect(() => {
  const timer = setInterval(() => {
    setCount(count + 1);
  }, 1000);
  return () => clearInterval(timer);
}, [count]); // effect re-subscribes when count changes

// ✅ FIX B: Functional update — doesn't need count in scope
useEffect(() => {
  const timer = setInterval(() => {
    setCount(prev => prev + 1); // prev is always current value
  }, 1000);
  return () => clearInterval(timer);
}, []); // empty deps OK — doesn't close over count

// ✅ FIX C: useRef for latest value without re-subscription
const countRef = useRef(count);
countRef.current = count;

useEffect(() => {
  const timer = setInterval(() => {
    console.log(countRef.current); // always current value
  }, 1000);
  return () => clearInterval(timer);
}, []);
```

### Pattern: Object/Array Identity Causing Infinite Loop

```typescript
// ❌ BROKEN: Infinite loop because options is a new object every render
function DataTable({ filters }) {
  const options = { pageSize: 10, filters }; // NEW OBJECT EVERY RENDER

  useEffect(() => {
    fetchData(options);
  }, [options]); // options changes every render → infinite loop
}

// ✅ FIX A: useMemo to stabilize object identity
function DataTable({ filters }) {
  const options = useMemo(() => ({
    pageSize: 10,
    filters,
  }), [filters]); // only new object when filters actually changes

  useEffect(() => {
    fetchData(options);
  }, [options]);
}

// ✅ FIX B: Use individual primitives as deps (best)
function DataTable({ filters }) {
  useEffect(() => {
    fetchData({ pageSize: 10, filters });
  }, [JSON.stringify(filters)]); // works but slightly hacky

  // BETTER: destructure filters to primitives
  const { search, category } = filters;
  useEffect(() => {
    fetchData({ pageSize: 10, filters: { search, category } });
  }, [search, category]); // primitives have stable identity
}

// ✅ FIX C: useRef for constant values
function DataTable({ filters }) {
  const optionsRef = useRef({ pageSize: 10, filters });
  
  useEffect(() => {
    optionsRef.current = { pageSize: 10, filters };
  }, [filters]);

  // For truly constant options:
  const OPTIONS = useRef({ pageSize: 10, sortBy: 'name' }).current;
}
```

### Pattern: Key Prop Bugs in Lists

```typescript
// ❌ BROKEN: Using index as key — React reuses wrong component instances
function TodoList({ todos }) {
  return todos.map((todo, index) => (
    <TodoItem key={index} todo={todo} /> // BUG when items are reordered/deleted
  ));
}
// When item at index 0 is deleted, all other items get new todo prop
// but React thinks they're the same component → stale state in children

// ✅ FIX: Use stable, unique IDs
function TodoList({ todos }) {
  return todos.map((todo) => (
    <TodoItem key={todo.id} todo={todo} /> // stable identity
  ));
}

// When stable IDs don't exist: generate them at data creation time
// NEVER generate IDs during render (causes remounting every render)
const [todos, setTodos] = useState(
  rawTodos.map(t => ({ ...t, id: crypto.randomUUID() })) // generate once
);

// For truly dynamic lists without IDs, use a combination:
function DynamicList({ items }) {
  const itemsWithKeys = useMemo(() => 
    items.map((item, index) => ({ 
      ...item, 
      key: item.id ?? `item-${index}-${hash(item)}` 
    })),
    [items]
  );
  
  return itemsWithKeys.map(item => (
    <ListItem key={item.key} {...item} />
  ));
}
```

### Pattern: useCallback Dependency Issues

```typescript
// ❌ BROKEN: callback changes on every render due to object dependency
function Form({ onSubmit }) {
  const [values, setValues] = useState({});
  
  const handleSubmit = useCallback(() => {
    onSubmit(values);
  }, [values]); // new object every render = new callback every render
  
  // Child re-renders unnecessarily because handleSubmit identity changes
  return <ExpensiveChild onClick={handleSubmit} />;
}

// ✅ FIX A: Use functional updates to avoid dependency
function Form({ onSubmit }) {
  const valuesRef = useRef(values);
  valuesRef.current = values;
  
  const handleSubmit = useCallback(() => {
    onSubmit(valuesRef.current);
  }, [onSubmit]); // stable
  
  return <ExpensiveChild onClick={handleSubmit} />;
}

// ✅ FIX B: Move state into custom hook
function useForm<T>(onSubmit: (values: T) => void) {
  const [values, setValues] = useState<T>({} as T);
  
  const handleSubmit = useCallback(() => {
    onSubmit(values);
  }, [values, onSubmit]);
  
  return { values, setValues, handleSubmit };
}
```

> **VIP EXPERT NOTE:** React bugs often stem from misunderstanding closures and 
> referential equality. When debugging: check your dependency arrays first. 
> Use the ESLint `react-hooks/exhaustive-deps` rule and never ignore it without 
> documenting why with a comment explaining the intentional exclusion.

---

## 🔷 FIX-04: TypeScript Type Safety

### Pattern: Remove All `any` Usage

```typescript
// ❌ BROKEN: `any` disables TypeScript — bugs hide until runtime
function processData(data: any) {
  return data.value.toUpperCase(); // no safety — crashes if value is null/number/missing
}

// ✅ FIX STEP 1: Use `unknown` — forces you to handle all types
function processData(data: unknown): string {
  // TypeScript forces you to narrow the type before using it
  if (typeof data !== 'object' || data === null) {
    throw new TypeError(`Expected object, got ${typeof data}`);
  }
  if (!('value' in data) || typeof (data as Record<string, unknown>).value !== 'string') {
    throw new TypeError(`Expected { value: string }, got ${JSON.stringify(data)}`);
  }
  return (data as { value: string }).value.toUpperCase(); // TypeScript now knows value is string
}

// ✅ FIX STEP 2 (Better): Define the exact type
interface DataPayload {
  value: string;
  metadata?: Record<string, unknown>;
}

function processData(data: DataPayload): string {
  return data.value.toUpperCase();
}

// Type guard for runtime validation
function isDataPayload(value: unknown): value is DataPayload {
  return (
    typeof value === 'object' &&
    value !== null &&
    'value' in value &&
    typeof (value as Record<string, unknown>).value === 'string'
  );
}
```

### Pattern: Non-Exhaustive Type Narrowing

```typescript
// ❌ BROKEN: New union member added but switch not updated — silent runtime error
type Status = 'pending' | 'active' | 'inactive' | 'banned'; // 'banned' added later

function getStatusColor(status: Status): string {
  switch (status) {
    case 'pending': return 'yellow';
    case 'active': return 'green';
    case 'inactive': return 'gray';
    // 'banned' is not handled! Returns undefined silently
  }
}

// ✅ FIX: Exhaustive switch with never check
function getStatusColor(status: Status): string {
  switch (status) {
    case 'pending': return 'yellow';
    case 'active': return 'green';
    case 'inactive': return 'gray';
    case 'banned': return 'red';
    default: {
      // TypeScript will error here if any case is unhandled
      const exhaustiveCheck: never = status;
      throw new Error(`Unhandled status: ${exhaustiveCheck}`);
    }
  }
}
// Now: add 'suspended' to Status → TypeScript errors at compile time

// ✅ FIX (Helper): Exhaustive check utility
function assertNever(value: never, message?: string): never {
  throw new Error(message ?? `Unhandled case: ${JSON.stringify(value)}`);
}

function getStatusColor(status: Status): string {
  switch (status) {
    case 'pending': return 'yellow';
    case 'active': return 'green';
    case 'inactive': return 'gray';
    case 'banned': return 'red';
    default:
      return assertNever(status);
  }
}
```

### Pattern: Strict Null Checks Violations

```typescript
// ❌ BROKEN: optional chaining abuse masks design issues
interface User {
  profile?: {
    name: string;
    email: string;
  };
}

function greet(user: User): string {
  return `Hello ${user.profile?.name ?? 'Anonymous'}`; // What if profile should always exist?
}

// ✅ FIX: Design types to make impossible states unrepresentable
type User = 
  | { status: 'anonymous' }
  | { status: 'authenticated'; profile: { name: string; email: string } };

function greet(user: User): string {
  if (user.status === 'anonymous') {
    return 'Hello Guest';
  }
  return `Hello ${user.profile.name}`; // TypeScript knows profile exists
}

// ✅ FIX (Records): Non-empty array guarantee
type NonEmptyArray<T> = [T, ...T[]];

function processBatch<T>(items: NonEmptyArray<T>): T {
  const [first, ...rest] = items;
  // first is always T, never undefined
  return first;
}

// Callers must ensure non-empty:
if (items.length > 0) {
  processBatch(items as NonEmptyArray<typeof items[0]>);
}
```

> **VIP EXPERT NOTE:** TypeScript's strict mode exists for a reason. 
> Enable: `strict: true`, `noImplicitAny: true`, `strictNullChecks: true`.
> The time "saved" by using `any` is repaid 10x in debugging production crashes.
> Treat every `any` or `@ts-ignore` as technical debt requiring a ticket to fix.

---

## 🔷 FIX-05: State Management Bugs

### Pattern: Inconsistent State

```typescript
// ❌ BROKEN: Multiple pieces of state can become inconsistent
const [isLoading, setIsLoading] = useState(false);
const [data, setData] = useState(null);
const [error, setError] = useState(null);
// Bug: What if isLoading=true AND error is set? That's impossible but can happen

// ✅ FIX: Use discriminated union state
type FetchState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

function useUserData(userId: string) {
  const [state, setState] = useState<FetchState<User>>({ status: 'idle' });

  useEffect(() => {
    setState({ status: 'loading' });

    fetchUser(userId)
      .then((data) => setState({ status: 'success', data }))
      .catch((error) => setState({
        status: 'error',
        error: error instanceof Error ? error : new Error(String(error))
      }));
  }, [userId]);

  return state;
}

// Usage: TypeScript forces handling all states
function UserDisplay({ userId }) {
  const state = useUserData(userId);

  if (state.status === 'idle') return null;
  if (state.status === 'loading') return <Spinner />;
  if (state.status === 'error') return <Error message={state.error.message} />;
  return <UserCard user={state.data} />; // TypeScript knows data is User here
}
```

### Pattern: State Update Race Conditions

```typescript
// ❌ BROKEN: Multiple rapid updates can arrive out of order
function Counter() {
  const [count, setCount] = useState(0);
  
  const increment = async () => {
    await saveToServer(count + 1);
    setCount(count + 1); // uses stale count if called rapidly
  };
}

// ✅ FIX: Functional updates for dependent state
const increment = async () => {
  setCount(prev => prev + 1); // always uses current value
  await saveToServer(count + 1); // still need to be careful here
};

// ✅ FIX: Sync state and server with transaction pattern
function useOptimisticCounter() {
  const [count, setCount] = useState(0);
  const [pending, setPending] = useState(0);
  
  const increment = useCallback(async () => {
    setPending(p => p + 1);
    const newCount = count + pending + 1;
    
    try {
      await saveToServer(newCount);
      setCount(newCount);
    } finally {
      setPending(p => p - 1);
    }
  }, [count, pending]);
  
  return { count: count + pending, increment };
}
```

> **VIP EXPERT NOTE:** State bugs are the hardest to debug because they involve 
> timing and ordering. Use the Reducer pattern for complex state, 
> immer for immutable updates, and always consider what happens if 
> two state updates occur simultaneously.

---

## 🔷 FIX-06: Data Transformation Bugs

### Pattern: In-Place Mutation

```typescript
// ❌ BROKEN: Mutates original array — callers see unexpected changes
function sortByName(users: User[]): User[] {
  return users.sort((a, b) => a.name.localeCompare(b.name)); // sort() MUTATES in place!
}
// Original users array is now sorted — this breaks any code relying on original order

// ✅ FIX: Always copy before mutating
function sortByName(users: User[]): User[] {
  return [...users].sort((a, b) => a.name.localeCompare(b.name));
  // OR
  return users.toSorted((a, b) => a.name.localeCompare(b.name)); // ES2023, non-mutating
}

// Complete list of Array methods that MUTATE (always copy first):
// push, pop, shift, unshift, sort, reverse, splice, fill, copyWithin
// Safe non-mutating alternatives:
// concat/spread, toSorted, filter, map, slice, reduce

// Object mutation protection:
function updateUser(user: User, updates: Partial<User>): User {
  // ❌ BROKEN: mutates original
  // Object.assign(user, updates);
  
  // ✅ CORRECT: returns new object
  return { ...user, ...updates };
}

// Deep update with immutable pattern:
function updateNested(state: State, path: string[], value: unknown): State {
  if (path.length === 0) return value as State;
  
  const [head, ...tail] = path;
  return {
    ...state,
    [head]: updateNested(state[head as keyof State] as State, tail, value)
  };
}
```

### Pattern: Float Precision Errors in Money/Finance

```typescript
// ❌ BROKEN: Floating point imprecision
const total = 0.1 + 0.2; // 0.30000000000000004
const price = 19.99;
const tax = price * 0.08; // 1.5992 — not 1.60

// ✅ FIX: Work in integer cents, format only for display
class Money {
  private readonly cents: number;

  constructor(cents: number) {
    if (!Number.isInteger(cents)) throw new TypeError('Money must be integer cents');
    this.cents = cents;
  }

  static fromDollars(dollars: number): Money {
    return new Money(Math.round(dollars * 100));
  }

  static fromCents(cents: number): Money {
    return new Money(cents);
  }

  add(other: Money): Money {
    return new Money(this.cents + other.cents);
  }

  subtract(other: Money): Money {
    return new Money(this.cents - other.cents);
  }

  multiply(factor: number): Money {
    return new Money(Math.round(this.cents * factor));
  }

  divide(divisor: number): Money {
    return new Money(Math.round(this.cents / divisor));
  }

  format(): string {
    return `$${(this.cents / 100).toFixed(2)}`;
  }

  toCents(): number {
    return this.cents;
  }

  toDollars(): number {
    return this.cents / 100;
  }
}

const price = Money.fromDollars(19.99);
const tax = price.multiply(0.08);
const total = price.add(tax);
console.log(total.format()); // "$21.59" — correct
```

### Pattern: Date Handling Bugs

```typescript
// ❌ BROKEN: Parsing dates is inconsistent across browsers
const date = new Date('2024-01-01'); // interpreted as UTC or local?

// ✅ FIX: Explicit timezone handling
import { format, parseISO, isValid } from 'date-fns';
import { zonedTimeToUtc, utcToZonedTime } from 'date-fns-tz';

// Parse ISO string to Date
function parseDate(dateString: string): Date | null {
  const parsed = parseISO(dateString);
  return isValid(parsed) ? parsed : null;
}

// Format for display in user's timezone
function formatForDisplay(date: Date, timezone: string): string {
  const zoned = utcToZonedTime(date, timezone);
  return format(zoned, 'MMM d, yyyy h:mm a');
}

// Store in UTC, display in local
const userTimezone = Intl.DateTimeFormat().resolvedOptions().timeZone;
const utcDate = zonedTimeToUtc('2024-01-15 14:30', userTimezone);
```

> **VIP EXPERT NOTE:** Data transformation bugs often manifest as "works on my machine" 
> issues. Always use immutable operations, never trust floating-point for money, 
> and be explicit about timezones. Write unit tests with edge cases: empty arrays, 
> single items, maximum values, and boundary conditions.

---

## 🔷 FIX-07: Security Bugs (SQL Injection, XSS, Secrets)

### Pattern: SQL Injection

```typescript
// ❌ CRITICAL VULNERABILITY: SQL injection
async function getUserByName(name: string) {
  const query = `SELECT * FROM users WHERE name = '${name}'`;
  // name = "'; DROP TABLE users; --" → database destroyed
  return db.query(query);
}

// ✅ FIX: Parameterized queries — ALWAYS
async function getUserByName(name: string): Promise<User | null> {
  const result = await db.query<User>(
    'SELECT id, name, email FROM users WHERE name = $1 LIMIT 1',
    [name] // parameter binding — name is never interpreted as SQL
  );
  return result.rows[0] ?? null;
}

// ✅ FIX: Query builder with automatic escaping
import { sql } from './db';

async function searchUsers(searchTerm: string) {
  // sql tagged template handles escaping automatically
  const result = await sql<User>`
    SELECT * FROM users 
    WHERE name ILIKE ${`%${searchTerm}%`}
    LIMIT 100
  `;
  return result;
}

// ✅ FIX: ORM with parameter binding
async function getUserByEmail(email: string) {
  return prisma.user.findUnique({
    where: { email }, // automatically parameterized
  });
}
```

### Pattern: XSS (Cross-Site Scripting)

```typescript
// ❌ CRITICAL VULNERABILITY: Injects raw HTML
function renderUserBio(bio: string) {
  container.innerHTML = bio; // if bio = '<script>stealCookies()</script>' → executed
}

// ✅ FIX A: Use textContent for plain text
function renderUserBio(bio: string) {
  container.textContent = bio; // always escaped, never interpreted as HTML
}

// ✅ FIX B: Sanitize if HTML is required (use DOMPurify)
import DOMPurify from 'dompurify';
function renderUserBio(bio: string) {
  container.innerHTML = DOMPurify.sanitize(bio, { 
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a'],
    ALLOWED_ATTR: ['href'],
  });
}

// React automatically escapes content, but watch out for:
// ❌ BROKEN:
function DangerousComponent({ htmlContent }) {
  return <div dangerouslySetInnerHTML={{ __html: htmlContent }} />;
}

// ✅ CORRECT:
import DOMPurify from 'isomorphic-dompurify';

function SafeComponent({ htmlContent }) {
  const sanitized = DOMPurify.sanitize(htmlContent);
  return <div dangerouslySetInnerHTML={{ __html: sanitized }} />;
}

// URL validation for user-provided links
function isSafeUrl(url: string): boolean {
  try {
    const parsed = new URL(url);
    return ['http:', 'https:'].includes(parsed.protocol);
  } catch {
    return false;
  }
}
```

### Pattern: Secrets Exposure

```typescript
// ❌ CRITICAL: Hardcoded secret in source code
const API_KEY = 'sk-live-abc123def456'; // exposed in git history forever

// ❌ CRITICAL: Secret logged
logger.info('API call', { url, apiKey: process.env.API_KEY }); // logged to observability

// ✅ FIX: Environment variables + validation + never log
const config = {
  apiKey: process.env.API_KEY,
  apiUrl: process.env.API_URL,
} as const;

// Validate at startup
if (!config.apiKey) throw new Error('API_KEY environment variable is required');

// Never log secrets — log redacted version
function redactSensitive(data: Record<string, unknown>): Record<string, unknown> {
  const sensitiveKeys = ['apiKey', 'password', 'token', 'secret', 'authorization'];
  return Object.entries(data).reduce((acc, [key, value]) => {
    const isSensitive = sensitiveKeys.some(sk => key.toLowerCase().includes(sk));
    acc[key] = isSensitive ? '[REDACTED]' : value;
    return acc;
  }, {} as Record<string, unknown>);
}

logger.info('API call', redactSensitive({ url, apiKey: config.apiKey }));
```

### Pattern: Path Traversal

```typescript
// ❌ CRITICAL VULNERABILITY: Path traversal attack
app.get('/download', (req, res) => {
  const filePath = path.join('/uploads', req.query.filename);
  res.sendFile(filePath);
  // filename = '../../../etc/passwd' → reads system files
});

// ✅ FIX: Validate and sanitize file paths
import path from 'path';

const UPLOAD_DIR = path.resolve('/app/uploads');

function getSafeFilePath(userInput: string): string | null {
  // Remove null bytes
  const sanitized = userInput.replace(/\0/g, '');
  
  // Join with base directory and resolve
  const fullPath = path.resolve(UPLOAD_DIR, sanitized);
  
  // Ensure the resolved path is within the allowed directory
  if (!fullPath.startsWith(UPLOAD_DIR)) {
    return null; // Path traversal attempt detected
  }
  
  return fullPath;
}

app.get('/download', (req, res) => {
  const filePath = getSafeFilePath(req.query.filename);
  if (!filePath) {
    return res.status(400).json({ error: 'Invalid filename' });
  }
  res.sendFile(filePath);
});
```

> **VIP EXPERT NOTE:** Security bugs have no "minor" severity. A single XSS or SQL 
> injection can compromise your entire system. Use automated security scanning 
> (Snyk, Dependabot), never disable security lint rules, and treat every user 
> input as hostile. When in doubt, sanitize, parameterize, and validate.

---

## 🔷 FIX-08: Python-Specific Patterns

### Pattern: Mutable Default Arguments

```python
# ❌ BROKEN: Default list is created ONCE at function definition
def add_item(item: str, items: list = []) -> list:
    items.append(item)
    return items

add_item("apple")   # ['apple']
add_item("banana")  # ['apple', 'banana'] ← BUG! Shares same list
add_item("cherry")  # ['apple', 'banana', 'cherry']

# ✅ CORRECT: Use None as sentinel, create new object each call
def add_item(item: str, items: list[str] | None = None) -> list[str]:
    if items is None:
        items = []
    items.append(item)
    return items

# ✅ CORRECT: Alternative with explicit copy
def add_item(item: str, items: list[str] | None = None) -> list[str]:
    result = list(items) if items is not None else []
    result.append(item)
    return result
```

### Pattern: Context Manager for Resources

```python
# ❌ BROKEN: Resource leak if exception occurs
def read_and_process(filepath: str) -> str:
    f = open(filepath)
    data = f.read()    # if this raises → file never closed
    result = process(data)
    f.close()
    return result

# ✅ CORRECT: Context manager guarantees cleanup
from pathlib import Path

def read_and_process(filepath: Path) -> str:
    with filepath.open(encoding='utf-8') as f:
        data = f.read()
    return process(data)

# ✅ CORRECT: Multiple context managers
def transfer_data(source: Path, dest: Path) -> None:
    with (
        source.open('r', encoding='utf-8') as src,
        dest.open('w', encoding='utf-8') as dst
    ):
        dst.write(src.read())

# Custom context manager with contextlib
from contextlib import contextmanager

@contextmanager
def managed_connection(host: str):
    conn = create_connection(host)
    try:
        yield conn
    finally:
        conn.close()

with managed_connection('localhost') as conn:
    conn.execute("SELECT * FROM users")
```

### Pattern: Exception Handling Anti-Patterns

```python
# ❌ BROKEN: Bare except catches everything including KeyboardInterrupt
try:
    process_data()
except:  # catches SystemExit, KeyboardInterrupt — very bad
    pass

# ❌ BROKEN: Losing exception information
try:
    process_data()
except ValueError:
    raise CustomError("Processing failed")  # Original traceback lost!

# ✅ CORRECT: Catch specific exceptions with context
try:
    process_data()
except ValueError as e:
    logger.warning(f"Invalid data format: {e}")
    raise ProcessingError(f"Could not process: {e}") from e  # Preserves chain

# ✅ CORRECT: Exception groups (Python 3.11+)
try:
    with ExceptionGroup() as eg:
        eg.submit(task1)
        eg.submit(task2)
except* ValueError as eg:
    logger.error(f"Validation errors: {eg.exceptions}")
except* RuntimeError as eg:
    logger.error(f"Runtime errors: {eg.exceptions}")
```

### Pattern: String Formatting Security

```python
# ❌ BROKEN: SQL injection via string formatting
def get_user(name: str) -> dict:
    query = f"SELECT * FROM users WHERE name = '{name}'"
    return db.execute(query)

# ✅ CORRECT: Parameterized queries
def get_user(name: str) -> dict | None:
    query = "SELECT * FROM users WHERE name = %s"
    result = db.execute(query, (name,))
    return result.fetchone()

# ❌ BROKEN: Command injection
import os
def process_file(filename: str) -> None:
    os.system(f"convert {filename} output.png")

# ✅ CORRECT: Use subprocess with argument list
import subprocess
def process_file(filename: str) -> None:
    subprocess.run(
        ["convert", filename, "output.png"],
        capture_output=True,
        check=True
    )
```

> **VIP EXPERT NOTE:** Python's dynamic nature makes certain bugs more subtle. 
> Always use type hints (mypy), avoid mutable defaults, use context managers 
> religiously, and be aware that `except:` catches KeyboardInterrupt. 
> Consider `pydantic` for runtime validation of external data.

---

## 🔷 FIX-09: Rate Limiting & Input Throttling

### Pattern: API Rate Limiting

```typescript
// ❌ BROKEN: No rate limiting — vulnerable to abuse and DDoS
app.post('/api/login', async (req, res) => {
  const result = await authenticate(req.body);
  return res.json(result);
});

// ✅ FIX: Rate limiting with sliding window
import rateLimit from 'express-rate-limit';

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5,                    // 5 attempts per window
  message: { error: 'Too many login attempts. Try again in 15 minutes.' },
  standardHeaders: true,     // Return rate limit info in headers
  legacyHeaders: false,
  keyGenerator: (req) => req.ip + ':' + req.body.email, // per IP + email combo
  skipSuccessfulRequests: false,
  handler: (req, res, next, options) => {
    logger.warn('Rate limit exceeded', { 
      ip: req.ip, 
      path: req.path,
      userAgent: req.get('user-agent')
    });
    res.status(429).json(options.message);
  }
});

app.post('/api/login', loginLimiter, async (req, res) => {
  const result = await authenticate(req.body);
  return res.json(result);
});

// Distributed rate limiting with Redis
import Redis from 'ioredis';
const redis = new Redis();

async function checkRateLimit(key: string, limit: number, windowSeconds: number): Promise<boolean> {
  const now = Date.now();
  const windowStart = now - (windowSeconds * 1000);
  
  const pipeline = redis.pipeline();
  pipeline.zremrangebyscore(key, 0, windowStart);  // Remove old entries
  pipeline.zcard(key);                             // Count current entries
  pipeline.zadd(key, now, `${now}-${Math.random()}`);  // Add current request
  pipeline.pexpire(key, windowSeconds * 1000);     // Set expiry
  
  const results = await pipeline.exec();
  const currentCount = results?.[1]?.[1] as number ?? 0;
  
  return currentCount < limit;
}
```

### Pattern: Frontend Debounce for API Calls

```typescript
// ❌ BROKEN: Fires API call on every keystroke
function SearchInput() {
  const [query, setQuery] = useState('');
  useEffect(() => {
    if (query) searchApi(query).then(setResults); // 20+ API calls for "hello world"
  }, [query]);
}

// ✅ FIX: Debounce input before API call
function SearchInput() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<SearchResult[]>([]);
  
  // Debounce: only fire API call after user stops typing for 300ms
  const debouncedQuery = useDebouncedValue(query, 300);
  
  useEffect(() => {
    if (!debouncedQuery) { setResults([]); return; }
    const controller = new AbortController();
    searchApi(debouncedQuery, { signal: controller.signal })
      .then(setResults)
      .catch(err => { if (err.name !== 'AbortError') console.error(err); });
    return () => controller.abort();
  }, [debouncedQuery]);

  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}

// useDebouncedValue hook
function useDebouncedValue<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);
  
  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);
  
  return debouncedValue;
}

// ✅ FIX: Throttle for scroll/resize events
function useThrottledCallback<T extends (...args: unknown[]) => unknown>(
  callback: T,
  limit: number
): T {
  const lastCall = useRef(0);
  
  return useCallback((...args: Parameters<T>) => {
    const now = Date.now();
    if (now - lastCall.current >= limit) {
      lastCall.current = now;
      callback(...args);
    }
  }, [callback, limit]) as T;
}
```

> **VIP EXPERT NOTE:** Rate limiting is both a security measure and a performance 
> optimization. Implement it at multiple layers: edge (CDN), gateway (nginx), 
> application (middleware), and database (query limits). For frontend, always 
> debounce user input that triggers API calls — it improves UX and reduces server load.

---

## 🔷 FIX-10: CORS Configuration

```typescript
// ❌ BROKEN: Wildcard CORS in production — any site can call your API
app.use(cors({ origin: '*' })); // NEVER in production

// ❌ BROKEN: Reflecting origin — allows any site to bypass CORS
app.use(cors({ origin: (origin, cb) => cb(null, origin || true) }));

// ✅ FIX: Whitelist specific origins
const ALLOWED_ORIGINS = [
  'https://myapp.com',
  'https://admin.myapp.com',
  ...(process.env.NODE_ENV === 'development' ? ['http://localhost:3000'] : []),
];

app.use(cors({
  origin: (origin, callback) => {
    // Allow requests with no origin (mobile apps, curl, Postman)
    if (!origin || ALLOWED_ORIGINS.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error(`CORS: Origin ${origin} not allowed`));
    }
  },
  credentials: true,          // Allow cookies
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Request-ID'],
  exposedHeaders: ['X-RateLimit-Remaining', 'X-Request-ID'],
  maxAge: 86400,              // Cache preflight for 24h
}));

// Environment-specific CORS configuration
function getCorsOptions(): cors.CorsOptions {
  const isDev = process.env.NODE_ENV === 'development';
  
  return {
    origin: isDev 
      ? true  // Allow all origins in development
      : (origin, callback) => {
          const allowed = process.env.ALLOWED_ORIGINS?.split(',') ?? [];
          if (!origin || allowed.includes(origin)) {
            callback(null, true);
          } else {
            callback(new Error('Not allowed by CORS'));
          }
        },
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'],
  };
}
```

> **VIP EXPERT NOTE:** Misconfigured CORS is a common vulnerability. Never use 
> wildcard origins in production, never reflect the request origin back, and 
> always validate origins against an explicit whitelist. CORS is not a security 
> mechanism — it's a relaxation of the same-origin policy. Always authenticate 
> requests regardless of CORS configuration.

---

## 🔷 FIX-11: Environment Variable Validation (Startup Safety)

```typescript
// ❌ BROKEN: App starts and crashes later when env var is used
const apiKey = process.env.API_KEY; // might be undefined
// ... 500 lines later ...
fetch(url, { headers: { Authorization: apiKey } }); // undefined → silent failure

// ✅ FIX: Validate ALL required env vars at startup — fail fast
import { z } from 'zod';

const EnvSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']),
  PORT: z.string().transform(Number).pipe(z.number().positive()),
  DATABASE_URL: z.string().url(),
  API_KEY: z.string().min(10),
  JWT_SECRET: z.string().min(32),
  REDIS_URL: z.string().url().optional(),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
});

// Call this ONCE at application startup
function validateEnv(): z.infer<typeof EnvSchema> {
  const result = EnvSchema.safeParse(process.env);
  if (!result.success) {
    console.error('❌ Environment validation failed:');
    for (const issue of result.error.issues) {
      console.error(`  ${issue.path.join('.')}: ${issue.message}`);
    }
    process.exit(1); // Fail fast — don't start with bad config
  }
  return result.data;
}

export const env = validateEnv();
// Now env.API_KEY is guaranteed to be a string, never undefined

// Type-safe config module
export const config = {
  server: {
    port: env.PORT,
    env: env.NODE_ENV,
    logLevel: env.LOG_LEVEL,
  },
  database: {
    url: env.DATABASE_URL,
  },
  auth: {
    jwtSecret: env.JWT_SECRET,
    apiKey: env.API_KEY,
  },
  cache: {
    redisUrl: env.REDIS_URL,
  },
} as const;
```

> **VIP EXPERT NOTE:** This is the single most impactful pattern for production
> reliability. 30% of "mysterious production crashes" are missing environment
> variables. Validate at startup, fail fast, provide clear error messages.
> Never allow the application to start with invalid configuration — it will 
> fail at the worst possible time (under load).

---

## 🔷 FIX-12: Memory Leaks

### Pattern: Event Listener Accumulation

```typescript
// ❌ BROKEN: Event listeners accumulate on every render
function DataStream() {
  useEffect(() => {
    const handler = (data) => setData(data);
    eventBus.on('data', handler); // Never removed!
  });
}

// ✅ FIX: Always remove event listeners in cleanup
function DataStream() {
  useEffect(() => {
    const handler = (data) => setData(data);
    eventBus.on('data', handler);
    
    return () => {
      eventBus.off('data', handler);
    };
  }, []);
}

// ✅ FIX: Use hook abstraction
function useEventBus(event: string, handler: (data: unknown) => void) {
  useEffect(() => {
    eventBus.on(event, handler);
    return () => eventBus.off(event, handler);
  }, [event, handler]);
}
```

### Pattern: Subscription Memory Leaks

```typescript
// ❌ BROKEN: Subscriptions not cleaned up
function useWebSocket(url: string) {
  const [messages, setMessages] = useState([]);
  
  useEffect(() => {
    const ws = new WebSocket(url);
    ws.onmessage = (e) => setMessages(m => [...m, e.data]);
    // ws never closed!
  }, [url]);
  
  return messages;
}

// ✅ FIX: Proper cleanup
function useWebSocket(url: string) {
  const [messages, setMessages] = useState([]);
  
  useEffect(() => {
    const ws = new WebSocket(url);
    ws.onmessage = (e) => setMessages(m => [...m, e.data]);
    
    return () => {
      if (ws.readyState === WebSocket.OPEN) {
        ws.close();
      }
    };
  }, [url]);
  
  return messages;
}
```

> **VIP EXPERT NOTE:** Memory leaks often manifest as "app gets slower over time."
> Use Chrome DevTools Memory tab to profile heap usage, watch for event listeners 
> and DOM nodes accumulating, and always implement cleanup functions in useEffect.

---

## 🔷 FIX VERIFICATION CHECKLIST

After applying any fix, verify:

```
□ Test written that FAILS on old code, PASSES on new code
□ Root cause documented in code comment or commit message
□ No new TypeScript errors introduced
□ No new linter warnings
□ Adjacent code still works
□ Performance not degraded
□ No new security vulnerabilities
□ Edge cases handled (null, empty, boundary)
□ Error messages are clear and actionable
□ Documentation updated if behavior changed
□ Memory profile checked (for leak fixes)
□ Backwards compatibility considered (for API changes)
```

---

*This module provides proven fix patterns. Always write tests before fixing bugs.*
