# AI Coding Anti-Patterns: What NOT to Do

## 🎯 Purpose

Catalog of common AI coding mistakes with correct alternatives. These patterns are observed frequently in AI-generated code and must be actively avoided.

> **VIP EXPERT NOTE — The AI Code Smell Radar:**
> AI-generated code has *distinctive patterns* of failure. The most dangerous:
> 1. **Plausible but wrong** — looks correct, compiles, even passes basic tests, but has a subtle logic flaw
> 2. **Over-solution** — more complex than needed, adding abstractions nobody asked for
> 3. **Copy-drift** — copies a pattern from training data that doesn't fit the current context
> 4. **Silent failure** — catches errors without handling them, returns undefined, swallows nulls
> Always review AI code with these failure modes in mind.

---

## 🔷 CATEGORY 1: ERROR HANDLING ANTI-PATTERNS

> **VIP EXPERT NOTE — Error Handling Philosophy:**
> Errors are not exceptional — they are expected. A robust system is defined
> not by how it handles success, but by how it handles failure. Every error
> should be visible, actionable, and contextual.

### Anti-Pattern 1.1: Silent Error Swallowing

```typescript
// ❌ WRONG: Error is caught and silently discarded
async function fetchData(id: string) {
  try {
    const response = await fetch(`/api/data/${id}`);
    return response.json();
  } catch (e) {
    // Silent swallow — error is lost forever
  }
}

// ❌ WRONG: Logging without handling
async function fetchData(id: string) {
  try {
    const response = await fetch(`/api/data/${id}`);
    return response.json();
  } catch (e) {
    console.log(e); // Logged but not handled, caller gets undefined
  }
}

// ✅ CORRECT: Explicit error handling
async function fetchData(id: string): Promise<Data | null> {
  try {
    const response = await fetch(`/api/data/${id}`);
    if (!response.ok) {
      throw new ApiError(response.status, `Failed to fetch data ${id}`);
    }
    return response.json();
  } catch (error) {
    if (error instanceof ApiError && error.status === 404) {
      return null; // Explicit: 404 means not found
    }
    // Re-throw unexpected errors
    throw new DataFetchError(`Failed to fetch data ${id}`, { cause: error });
  }
}
```

> **VIP EXPERT NOTE — The Empty Catch Block:**
> Empty catch blocks are a code smell of the highest order. If you truly
> want to ignore an error, comment WHY. Otherwise, you're just hiding bugs.

### Anti-Pattern 1.2: Catching Too Broadly

```python
# ❌ WRONG: Catches everything including SystemExit, KeyboardInterrupt
def process_data(data):
    try:
        return transform(data)
    except:
        return None

# ❌ WRONG: Too broad, hides bugs
def process_data(data):
    try:
        return transform(data)
    except Exception:
        return None

# ✅ CORRECT: Catch specific exceptions
def process_data(data: dict) -> ProcessedData:
    try:
        return transform(data)
    except (KeyError, TypeError) as e:
        raise ValidationError(f"Invalid data format: {e}") from e
    except ValueError as e:
        raise DataError(f"Data validation failed: {e}") from e
```

> **VIP EXPERT NOTE — Exception Granularity:**
> Catching `Exception` masks bugs. Only catch what you can handle. In Python,
> bare `except:` catches SystemExit and KeyboardInterrupt — never do this.

### Anti-Pattern 1.3: Missing Error Context

```typescript
// ❌ WRONG: Generic error message
if (!user) {
  throw new Error('User not found');
}

// ✅ CORRECT: Include context for debugging
if (!user) {
  throw new UserNotFoundError(userId, {
    context: { 
      searchedId: userId, 
      timestamp: new Date().toISOString(),
      requestId: getRequestId(),
    }
  });
}

// ✅ CORRECT: Chain errors to preserve stack traces
class ServiceError extends Error {
  constructor(message: string, public readonly cause?: Error) {
    super(message);
    this.name = 'ServiceError';
  }
}

try {
  await database.query('SELECT * FROM users');
} catch (dbError) {
  throw new ServiceError('Failed to load users', dbError);
}
```

---

## 🔷 CATEGORY 2: TYPE SAFETY ANTI-PATTERNS

> **VIP EXPERT NOTE — Type Safety as Documentation:**
> Types are executable documentation. They tell you what values are valid,
> what operations are safe, and what invariants hold. Using `any` or unsafe
> assertions is like removing comments and hoping the code makes sense.

### Anti-Pattern 2.1: Using `any` Type

```typescript
// ❌ WRONG: any disables all type checking
function processData(data: any) {
  return data.value.name.toUpperCase();
}

// ✅ CORRECT: Define proper types
interface DataPayload {
  value: {
    name: string;
  };
}

function processData(data: DataPayload): string {
  return data.value.name.toUpperCase();
}

// ✅ CORRECT: Use unknown for truly unknown types
function processData(data: unknown): string {
  if (typeof data !== 'object' || data === null) {
    throw new TypeError('Expected object');
  }
  if (!('value' in data)) {
    throw new TypeError('Missing value property');
  }
  // ... continue narrowing
}
```

> **VIP EXPERT NOTE — Unknown vs Any:**
> `unknown` is the type-safe counterpart to `any`. It forces you to validate
> before using. Use `any` only when migrating legacy code — and even then,
> prefer `unknown` with type guards.

### Anti-Pattern 2.2: Type Assertion Without Validation

```typescript
// ❌ WRONG: Assertion without validation
const user = JSON.parse(jsonString) as User;

// ✅ CORRECT: Validate with schema
import { z } from 'zod';

const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  email: z.string().email(),
});

function parseUser(jsonString: string): User {
  const parsed = JSON.parse(jsonString);
  return UserSchema.parse(parsed); // Throws with detailed error if invalid
}

// ✅ CORRECT: Type guard function
function isUser(obj: unknown): obj is User {
  return (
    typeof obj === 'object' &&
    obj !== null &&
    'id' in obj &&
    typeof (obj as User).id === 'string' &&
    'name' in obj &&
    typeof (obj as User).name === 'string'
  );
}

const parsed = JSON.parse(jsonString);
if (!isUser(parsed)) {
  throw new TypeError('Invalid user object');
}
// parsed is now typed as User
```

### Anti-Pattern 2.3: Non-Null Assertion Operator Abuse

```typescript
// ❌ WRONG: ! operator assumes value exists
function getUserName(user: User | null): string {
  return user!.name; // Crashes at runtime if user is null
}

// ✅ CORRECT: Handle null explicitly
function getUserName(user: User | null): string {
  if (!user) {
    return 'Unknown User';
  }
  return user.name;
}

// ✅ CORRECT: Use optional chaining
function getUserName(user: User | null): string {
  return user?.name ?? 'Unknown User';
}

// ✅ CORRECT: Require non-null at API boundary
function processUser(user: User): void {
  // Function signature guarantees user is not null
  console.log(user.name); // Safe
}

// Caller must validate
const user = await fetchUser(id);
if (!user) {
  throw new Error('User not found');
}
processUser(user); // Type-safe: user is guaranteed non-null here
```

> **VIP EXPERT NOTE — The Non-Null Lie:**
> `!` tells TypeScript "trust me, I know better." You're probably wrong.
> Every `!` is a potential runtime crash. Eliminate them through proper
> validation, not assertions.

---

## 🔷 CATEGORY 3: ASYNC ANTI-PATTERNS

> **VIP EXPERT NOTE — Async Complexity:**
> Async code is harder to reason about than sync code. Each `await` is a
> potential interleaving point where other code can run. Minimize async
> boundaries and always consider what happens between await statements.

### Anti-Pattern 3.1: Missing Await

```typescript
// ❌ WRONG: Promise not awaited
async function processUser(id: string) {
  const user = fetchUser(id); // Returns Promise, not User
  return user.name; // undefined — Promise has no .name
}

// ✅ CORRECT: Always await promises
async function processUser(id: string): Promise<string> {
  const user = await fetchUser(id);
  return user.name;
}

// ✅ CORRECT: ESLint rule to catch missing awaits
// .eslintrc:
{
  "rules": {
    "@typescript-eslint/no-floating-promises": "error"
  }
}
```

> **VIP EXPERT NOTE — Floating Promises:**
> A Promise without await or .catch() is a bug waiting to happen. Use
> @typescript-eslint/no-floating-promises to catch these at build time.

### Anti-Pattern 3.2: Sequential Awaits for Independent Operations

```typescript
// ❌ WRONG: Sequential awaits for independent operations
async function loadDashboard(userId: string) {
  const user = await fetchUser(userId);       // 200ms
  const orders = await fetchOrders(userId);   // 300ms
  const stats = await fetchStats(userId);     // 150ms
  // Total: 650ms
  return { user, orders, stats };
}

// ✅ CORRECT: Parallel execution
async function loadDashboard(userId: string) {
  const [user, orders, stats] = await Promise.all([
    fetchUser(userId),
    fetchOrders(userId),
    fetchStats(userId),
  ]);
  // Total: 300ms (longest single call)
  return { user, orders, stats };
}

// ✅ CORRECT: Named parallel execution for clarity
async function loadDashboard(userId: string) {
  const userPromise = fetchUser(userId);
  const ordersPromise = fetchOrders(userId);
  const statsPromise = fetchStats(userId);
  
  const user = await userPromise;
  const orders = await ordersPromise;
  const stats = await statsPromise;
  
  return { user, orders, stats };
}
```

### Anti-Pattern 3.3: Fire-and-Forget Promises

```typescript
// ❌ WRONG: Promise started but never awaited or caught
function logAnalytics(event: string) {
  sendAnalytics(event); // Fire-and-forget — errors are lost
}

// ✅ CORRECT: Handle promise explicitly
async function logAnalytics(event: string): Promise<void> {
  try {
    await sendAnalytics(event);
  } catch (error) {
    // Log locally but don't fail the main operation
    logger.warn('Analytics failed', { event, error });
  }
}

// ✅ CORRECT: If truly fire-and-forget, at least catch errors
function logAnalytics(event: string): void {
  sendAnalytics(event).catch(error => {
    logger.warn('Analytics failed', { event, error });
  });
}

// ✅ CORRECT: Use void operator to indicate intentional non-await
function logAnalytics(event: string): void {
  void sendAnalytics(event); // Explicit: we don't care about result
}
```

> **VIP EXPERT NOTE — Promise Discipline:**
> Every Promise must have a handler: await it, .catch() it, or void it.
> Unhandled rejections crash Node.js processes. They are never "safe to ignore."

---

## 🔷 CATEGORY 4: STATE MANAGEMENT ANTI-PATTERNS

> **VIP EXPERT NOTE — State Complexity:**
> State is the source of most bugs. The less state you have, the fewer bugs.
> Prefer computed values over stored state. Prefer immutability over mutation.

### Anti-Pattern 4.1: Direct State Mutation

```typescript
// ❌ WRONG: Mutating state directly
const [user, setUser] = useState<User>({ name: '', email: '' });

function updateName(name: string) {
  user.name = name; // Direct mutation — React won't re-render
}

// ✅ CORRECT: Create new state object
function updateName(name: string) {
  setUser(prev => ({ ...prev, name }));
}

// ✅ CORRECT: Use Immer for complex mutations
import produce from 'immer';

function updateNestedState() {
  setUser(prev => produce(prev, draft => {
    draft.address.city = 'New York';
    draft.preferences.notifications.email = true;
  }));
}
```

### Anti-Pattern 4.2: Stale Closure in useEffect

```typescript
// ❌ WRONG: Closure captures stale values
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const interval = setInterval(() => {
      setCount(count + 1); // count is always 0 (initial value)
    }, 1000);
    return () => clearInterval(interval);
  }, []); // Empty deps — count is captured as 0
  
  return <div>{count}</div>;
}

// ✅ CORRECT: Use functional update
useEffect(() => {
  const interval = setInterval(() => {
    setCount(prev => prev + 1); // prev is always current
  }, 1000);
  return () => clearInterval(interval);
}, []);

// ✅ CORRECT: Include all dependencies (with caution)
useEffect(() => {
  const interval = setInterval(() => {
    setCount(count + 1);
  }, 1000);
  return () => clearInterval(interval);
}, [count]); // Effect re-runs when count changes (may cause issues)
```

> **VIP EXPERT NOTE — The useEffect Dependency Lie:**
> `// eslint-disable-next-line react-hooks/exhaustive-deps` is a red flag.
> It usually means you're using stale closures. Fix the underlying issue,
> don't silence the warning.

### Anti-Pattern 4.3: Derived State in useState

```typescript
// ❌ WRONG: Storing derived state
function UserList({ users }: { users: User[] }) {
  const [activeUsers, setActiveUsers] = useState<User[]>([]);
  
  useEffect(() => {
    setActiveUsers(users.filter(u => u.isActive));
  }, [users]);
  
  // If users changes and then a user becomes active, derived state may be stale
}

// ✅ CORRECT: Compute derived values directly
function UserList({ users }: { users: User[] }) {
  const activeUsers = useMemo(
    () => users.filter(u => u.isActive),
    [users]
  );
  // Always reflects current state of users
}

// ✅ CORRECT: If expensive, useMemo; if cheap, compute inline
function UserList({ users }: { users: User[] }) {
  const activeUsers = users.filter(u => u.isActive); // Simple enough
  // ...
}
```

---

## 🔷 CATEGORY 5: REACT ANTI-PATTERNS

> **VIP EXPERT NOTE — React Mental Model:**
> React is declarative, not imperative. Components describe what should
> render given current props/state, not how to update the DOM. Fight the
> urge to use refs and effects when props and state will do.

### Anti-Pattern 5.1: Array Index as Key

```tsx
// ❌ WRONG: Index as key for dynamic lists
function TodoList({ todos }: { todos: Todo[] }) {
  return (
    <ul>
      {todos.map((todo, index) => (
        <TodoItem key={index} todo={todo} /> // Wrong when list changes
      ))}
    </ul>
  );
}

// ✅ CORRECT: Use stable unique IDs
function TodoList({ todos }: { todos: Todo[] }) {
  return (
    <ul>
      {todos.map((todo) => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
    </ul>
  );
}

// ⚠️ EXCEPTION: Index is OK for static lists that never reorder
function StaticMenu({ items }: { items: string[] }) {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={index}>{item}</li> // OK if items never change order
      ))}
    </ul>
  );
}
```

> **VIP EXPERT NOTE — The Key Prop:**
> Keys are not for styling — they're for React to identify elements across
> renders. Wrong keys cause: lost state, extra re-renders, broken animations,
> and subtle bugs with uncontrolled inputs.

### Anti-Pattern 5.2: useEffect for Element Events

```tsx
// ❌ WRONG: Using useEffect + addEventListener for events that belong on the element
function Component() {
  const buttonRef = useRef<HTMLButtonElement>(null);
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const button = buttonRef.current;
    if (!button) return;
    const handleClick = () => setCount(c => c + 1);
    button.addEventListener('click', handleClick);
    return () => button.removeEventListener('click', handleClick);
  }, []);
  
  return <button ref={buttonRef}>{count}</button>;
  // This is overly complex for a simple click handler
}

// ✅ CORRECT: Use JSX event handlers for element events
function Component() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// ⚠️ NOTE: useEffect + addEventListener IS correct for:
//   - Document-level events (keyboard shortcuts, scroll)
//   - Window events (resize, online/offline)
//   - Third-party DOM elements not managed by React
//   - Events that need to be added/removed based on state
```

### Anti-Pattern 5.3: Prop Drilling

```tsx
// ❌ WRONG: Passing props through multiple layers
function App() {
  const user = useAuth();
  return <Layout user={user} />;
}

function Layout({ user }: { user: User }) {
  return <Sidebar user={user} />;
}

function Sidebar({ user }: { user: User }) {
  return <UserMenu user={user} />; // user passed through Layout just to get here
}

// ✅ CORRECT: Use Context for deeply nested data
const UserContext = createContext<User | null>(null);

function App() {
  const user = useAuth();
  return (
    <UserContext.Provider value={user}>
      <Layout />
    </UserContext.Provider>
  );
}

function UserMenu() {
  const user = useContext(UserContext); // Direct access
}

// ✅ CORRECT: Component composition avoids prop drilling
function App() {
  const user = useAuth();
  return (
    <Layout sidebar={<UserMenu user={user} />} />
  );
}
```

> **VIP EXPERT NOTE — Context Performance:**
> Context causes re-renders for all consumers when value changes. For
> high-frequency updates, use state management libraries (Zustand, Jotai)
> or split context into smaller pieces.

---

## 🔷 CATEGORY 6: PYTHON ANTI-PATTERNS

> **VIP EXPERT NOTE — Pythonic Code:**
> Python has idioms that differ from other languages. "Explicit is better
> than implicit" applies to error handling. "There should be one obvious
> way to do it" discourages clever one-liners.

### Anti-Pattern 6.1: Mutable Default Arguments

```python
# ❌ WRONG: Default list is shared across all calls
def add_item(item: str, items: list = []) -> list:
    items.append(item)
    return items

add_item("a")  # ["a"]
add_item("b")  # ["a", "b"] — WRONG! Should be ["b"]

# ✅ CORRECT: Use None as sentinel
def add_item(item: str, items: list | None = None) -> list:
    if items is None:
        items = []
    items.append(item)
    return items

# ✅ CORRECT: Immutable default with conversion
def add_item(item: str, items: tuple = ()) -> list:
    return list(items) + [item]
```

> **VIP EXPERT NOTE — The Mutable Default Trap:**
> Python evaluates default arguments once at function definition, not at
> call time. Mutable defaults are shared state — a recipe for bugs.

### Anti-Pattern 6.2: Bare Except

```python
# ❌ WRONG: Catches everything including KeyboardInterrupt
try:
    risky_operation()
except:
    pass

# ✅ CORRECT: Catch specific exceptions
try:
    risky_operation()
except (ValueError, TypeError) as e:
    logger.error(f"Operation failed: {e}")
except Exception as e:
    logger.exception(f"Unexpected error: {e}")
    raise
```

### Anti-Pattern 6.3: String Concatenation for Paths

```python
# ❌ WRONG: OS-specific, error-prone
path = directory + "/" + filename + ".txt"

# ✅ CORRECT: Use pathlib
from pathlib import Path
path = Path(directory) / f"{filename}.txt"

# ✅ CORRECT: For URL paths, use urllib
from urllib.parse import urljoin
url = urljoin(base_url, endpoint)
```

### Anti-Pattern 6.4: Using `is` for Equality

```python
# ❌ WRONG: 'is' checks identity, not equality
if x is 5:  # Works by accident for small integers
    pass

# ✅ CORRECT: Use == for value comparison
if x == 5:
    pass

# ✅ CORRECT: Use 'is' only for None, True, False, and identity checks
if x is None:
    pass
```

> **VIP EXPERT NOTE — `is` vs `==`:**
> `is` checks object identity (same memory address). `==` checks equality.
> Python caches small integers, so `x is 5` might work — until it doesn't.

---

## 🔷 CATEGORY 7: SECURITY ANTI-PATTERNS

> **VIP EXPERT NOTE — Security First:**
> Security is not a feature you add later — it's a property of the system.
> Every input is hostile until proven otherwise. Never trust user data,
> external APIs, or environment configuration.

### Anti-Pattern 7.1: SQL Injection

```typescript
// ❌ CRITICAL VULNERABILITY
const query = `SELECT * FROM users WHERE id = '${userId}'`;

// ✅ CORRECT: Parameterized query
const query = 'SELECT * FROM users WHERE id = $1';
const result = await db.query(query, [userId]);

// ✅ CORRECT: Query builder with proper escaping
const result = await db.users.findUnique({
  where: { id: userId }
});

// ❌ CRITICAL: Never use string interpolation for table/column names
// If dynamic identifiers are needed, use allowlists:
const ALLOWED_COLUMNS = ['name', 'email', 'created_at'];
if (!ALLOWED_COLUMNS.includes(sortColumn)) {
  throw new Error('Invalid sort column');
}
```

> **VIP EXPERT NOTE — SQL Injection Reality:**
> SQL injection is still #1 on OWASP Top 10. Use ORMs, query builders,
> or parameterized queries. Never concatenate user input into SQL.

### Anti-Pattern 7.2: XSS via innerHTML

```typescript
// ❌ CRITICAL VULNERABILITY
element.innerHTML = userInput;

// ✅ CORRECT: Use textContent
element.textContent = userInput;

// ✅ CORRECT: Sanitize if HTML needed
import DOMPurify from 'dompurify';
element.innerHTML = DOMPurify.sanitize(userInput);

// ✅ CORRECT: React automatically escapes (unless dangerouslySetInnerHTML)
function SafeComponent({ userInput }: { userInput: string }) {
  return <div>{userInput}</div>; // Automatically escaped
}

// ❌ DANGEROUS: Only use with sanitized content
function DangerousComponent({ html }: { html: string }) {
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}
```

### Anti-Pattern 7.3: Hardcoded Secrets

```typescript
// ❌ CRITICAL VULNERABILITY
const API_KEY = 'sk-live-abc123def456';
const DB_PASSWORD = 'password123';

// ✅ CORRECT: Environment variables
const API_KEY = process.env.API_KEY;
const DB_PASSWORD = process.env.DB_PASSWORD;

// Validate at startup
if (!API_KEY) throw new Error('API_KEY environment variable required');

// ✅ CORRECT: Use secrets management service
import { SecretsManager } from '@aws-sdk/client-secrets-manager';

const secret = await secretsManager.getSecretValue({ SecretId: 'api-key' });
```

> **VIP EXPERT NOTE — Secret Management:**
> Hardcoded secrets end up in git history. Use environment variables for
> development, secret management services (AWS Secrets Manager, Azure Key Vault)
> for production. Never commit .env files with real secrets.

### Anti-Pattern 7.4: Insecure Deserialization

```python
# ❌ CRITICAL VULNERABILITY: Executes arbitrary code
import pickle
data = pickle.loads(user_input)

# ✅ CORRECT: Use JSON for untrusted data
import json
data = json.loads(user_input)

# ✅ CORRECT: If pickle needed, validate source
data = pickle.loads(user_input)  # Only if input is from trusted source
```

---

## 🔷 CATEGORY 8: PERFORMANCE ANTI-PATTERNS

> **VIP EXPERT NOTE — Performance Philosophy:**
> Premature optimization is evil. But known anti-patterns are not premature.
> N+1 queries, missing memoization, and blocking operations are always wrong.

### Anti-Pattern 8.1: N+1 Queries

```typescript
// ❌ WRONG: Query in loop
for (const user of users) {
  const profile = await db.getProfile(user.id); // N queries
}

// ✅ CORRECT: Batch query
const userIds = users.map(u => u.id);
const profiles = await db.getProfiles(userIds); // 1 query
const profileMap = new Map(profiles.map(p => [p.userId, p]));

// ✅ CORRECT: Use ORM with eager loading
const usersWithProfiles = await db.users.findMany({
  include: { profile: true }
});
```

### Anti-Pattern 8.2: Missing Memoization

```tsx
// ❌ WRONG: Expensive computation on every render
function ExpensiveList({ items }: { items: Item[] }) {
  const sorted = items.sort((a, b) => a.name.localeCompare(b.name)); // Runs every render
  return <List items={sorted} />;
}

// ✅ CORRECT: Memoize expensive computation
function ExpensiveList({ items }: { items: Item[] }) {
  const sorted = useMemo(
    () => [...items].sort((a, b) => a.name.localeCompare(b.name)),
    [items]
  );
  return <List items={sorted} />;
}

// ✅ CORRECT: Memoize callback functions
function SearchComponent() {
  const [query, setQuery] = useState('');
  
  const handleSearch = useCallback((searchTerm: string) => {
    setQuery(searchTerm);
    trackSearch(searchTerm);
  }, []);
  
  return <SearchInput onSearch={handleSearch} />;
}

// ✅ CORRECT: Memoize component (for expensive children)
const MemoizedExpensiveComponent = memo(ExpensiveComponent);
```

> **VIP EXPERT NOTE — Memoization Trade-offs:**
> Memoization has a cost — comparison and cache storage. Don't memoize
> everything. Memoize when: computation is expensive, referential equality
> matters (for child props), or profiling shows it's needed.

### Anti-Pattern 8.3: Inefficient List Operations

```python
# ❌ WRONG: O(n) lookup in loop
items = [1, 2, 3, 4, 5]
for x in large_list:
    if x in items:  # O(n) lookup
        process(x)

# ✅ CORRECT: Use set for O(1) lookup
item_set = set(items)
for x in large_list:
    if x in item_set:  # O(1) lookup
        process(x)

# ❌ WRONG: String concatenation in loop
result = ""
for item in items:
    result += str(item)  # Creates new string each time

# ✅ CORRECT: Use join
result = "".join(str(item) for item in items)
```

---

## 🔷 CATEGORY 9: OVER-ENGINEERING ANTI-PATTERNS

> **VIP EXPERT NOTE — Simplicity Principle:**
> "Simple" is not "easy." Simple means few moving parts, clear logic,
> minimal abstractions. Easy means familiar. The best code is boring code
> that any developer can understand and modify.

### Anti-Pattern 9.1: Premature Abstraction

```typescript
// ❌ WRONG: Creating abstractions before you need them
interface IUserRepositoryFactory {
  createRepository(type: string): IUserRepository;
}

interface IUserRepository {
  findById(id: string): Promise<IUser>;
}

interface IUser {
  getId(): string;
  getName(): string;
}

// ... 200 lines of interfaces for a CRUD app with one data source

// ✅ CORRECT: Start simple, abstract when you have 2+ implementations
type User = { id: string; name: string; email: string };

async function findUserById(id: string): Promise<User | null> {
  return db.users.findUnique({ where: { id } });
}

// Add interfaces WHEN you actually need a second implementation
// (e.g., when you add caching, or swap databases)
```

> **VIP EXPERT NOTE — The Rule of Three:**
> Don't abstract until you have **three** concrete uses. Two is coincidence.
> Three is a pattern worth abstracting.

### Anti-Pattern 9.2: Unnecessary Wrapper Functions

```typescript
// ❌ WRONG: Wrapper that adds no value
function getUser(id: string): Promise<User> {
  return fetchUser(id); // just calls another function — adds no logic
}

// ❌ WRONG: Wrapper class around a simple data structure
class UserName {
  private value: string;
  constructor(name: string) { this.value = name; }
  getValue(): string { return this.value; }
  toString(): string { return this.value; }
}
// Just use a string! Unless you need validation logic.

// ✅ CORRECT: Abstracting only when adding real value
class Email {
  private constructor(private readonly value: string) {}
  
  static create(input: string): Email {
    if (!input.includes('@')) throw new ValidationError('Invalid email format');
    return new Email(input.toLowerCase().trim());
  }
  
  toString(): string { return this.value; }
  equals(other: Email): boolean { return this.value === other.value; }
}
// This adds validation, normalization, and comparison — worth wrapping.
```

### Anti-Pattern 9.3: Over-Engineered State Management

```typescript
// ❌ WRONG: Redux for simple local state
const INCREMENT = 'INCREMENT';
const DECREMENT = 'DECREMENT';

function counterReducer(state = 0, action) {
  switch (action.type) {
    case INCREMENT: return state + 1;
    case DECREMENT: return state - 1;
    default: return state;
  }
}

// ✅ CORRECT: useState for local component state
function Counter() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount(c => c - 1)}>-</button>
      {count}
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}
```

---

## 🔷 CATEGORY 10: DEPENDENCY MANAGEMENT ANTI-PATTERNS

> **VIP EXPERT NOTE — Dependency Hygiene:**
> Every dependency is a liability. It can have bugs, security vulnerabilities,
> and breaking changes. Choose dependencies carefully, update regularly,
> and remove unused ones.

### Anti-Pattern 10.1: Importing Entire Libraries

```typescript
// ❌ WRONG: Importing entire lodash (70KB+)
import _ from 'lodash';
const unique = _.uniq(items);

// ✅ CORRECT: Import only what you need
import uniq from 'lodash/uniq';
const unique = uniq(items);

// ✅ BETTER: Use native JS when possible (no dependency needed!)
const unique = [...new Set(items)];

// ✅ CORRECT: Tree-shaking friendly imports
import { debounce, throttle } from 'lodash-es'; // ES modules version
```

### Anti-Pattern 10.2: Not Pinning Dependencies

```json
// ❌ WRONG: Version ranges can introduce breaking changes
{
  "dependencies": {
    "express": "^4.18.0",     // Could install 4.19.x with breaking changes
    "react": "*"              // Could install any version!
  }
}

// ✅ CORRECT: Pin exact versions + use lockfiles
{
  "dependencies": {
    "express": "4.18.2",      // Exact version
    "react": "18.2.0"         // Exact version
  }
}
// Always commit package-lock.json / yarn.lock / pnpm-lock.yaml
```

### Anti-Pattern 10.3: Ignoring Security Vulnerabilities

```bash
# ❌ WRONG: Never check for vulnerabilities
npm install

# ✅ CORRECT: Regular security audits
npm audit
npm audit fix

# ✅ CORRECT: Automated scanning in CI
# .github/workflows/security.yml
- name: Run security audit
  run: npm audit --audit-level=moderate
```

> **VIP EXPERT NOTE — Supply Chain Security:**
> npm packages have been compromised to steal credentials. Use lockfiles,
> audit regularly, and consider tools like Snyk or Dependabot for automated
> vulnerability alerts.

---

## 🔷 ANTI-PATTERN DETECTION CHECKLIST

Before submitting code, scan for these patterns:

```
□ No silent catch blocks
□ No `any` types without justification
□ No `!` non-null assertions without validation
□ No missing awaits on Promises
□ No direct state mutations
□ No array index as React key for dynamic lists
□ No mutable default arguments (Python)
□ No bare except clauses (Python)
□ No SQL string concatenation
□ No innerHTML with user input
□ No hardcoded secrets
□ No N+1 queries
□ No missing memoization for expensive computations
□ No premature abstractions
□ No unnecessary wrapper functions
□ No wildcard dependencies
□ No floating promises
□ No console.log in production code paths
```

---

## 🔷 VIP EXPERT NOTE: CODE REVIEW CHECKLIST

> Before submitting any code for review, mentally check:
> 
> 1. **Correctness**: Does it handle edge cases? Null inputs? Empty arrays?
> 2. **Error Handling**: Are all errors handled explicitly? No silent failures?
> 3. **Type Safety**: Are types as specific as possible? No `any` escapes?
> 4. **Performance**: Are there N+1 queries? Unnecessary re-renders?
> 5. **Security**: Is user input sanitized? No hardcoded secrets?
> 6. **Simplicity**: Can this be simpler? Is the abstraction justified?
> 7. **Testing**: Can this be tested? Are there obvious test cases?
> 
> A 5-minute self-review saves hours of back-and-forth in PR review.

---

*This module catalogs common AI coding mistakes. Always review generated code against these patterns.*
