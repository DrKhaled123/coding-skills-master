# 🩺 SKILL: Bug Diagnosis & Root Cause Analysis

## 🎯 Purpose

Systematically locate, isolate, and understand bugs before touching any code. This skill implements the **Binary Search Isolation** method combined with **Bug Taxonomy Classification** for rapid, accurate diagnosis.

> **VIP EXPERT NOTE — The 80/20 Rule of Debugging:**
> 80% of debugging time should be spent *understanding* the bug. 20% on fixing it.
> If you spend more time fixing than understanding, you're patching symptoms.
> The fix for a well-understood bug is almost always trivial (1-5 lines).

---

## 🔷 THE 6-PHASE DIAGNOSIS PROTOCOL

```
┌──────────────────────────────────────────────────────────────┐
│  PHASE 0: TRIAGE — Is this actually a bug? (30 seconds)      │
│  PHASE 1: REPRODUCE — Confirm the bug exists                 │
│  PHASE 2: ISOLATE — Narrow to smallest failing unit          │
│  PHASE 3: CATEGORIZE — Identify the bug type                 │
│  PHASE 4: ROOT CAUSE — Find the origin                       │
│  PHASE 5: DOCUMENT — Write the root cause statement          │
└──────────────────────────────────────────────────────────────┘
```

---

## Phase 0: TRIAGE — Is This Actually a Bug?

> **VIP EXPERT NOTE:** 15% of reported "bugs" are actually expected behavior, configuration issues, or stale caches. Spend 30 seconds triaging before diving in.

### Quick Triage Checklist
```
□ Is this expected behavior that the reporter misunderstands?
□ Has this ever worked? (If no → it's a missing feature, not a bug)
□ Is this a local-only issue? (stale cache, wrong branch, uncommitted changes)
□ Can I reproduce it in a clean environment? (incognito, fresh install)
□ Is there a known issue / recent deploy that explains it?
□ When did it first appear? (check git history, deploy logs)
```

**If all answers suggest it IS a real bug → Proceed to Phase 1**

---

## Phase 1: REPRODUCE — Confirm the Bug Exists

### Reproduction Checklist
```
□ Can you reproduce it consistently?
□ What are the exact steps to reproduce?
□ What is the exact input that causes it?
□ What is the expected output?
□ What is the actual output?
□ Is it environment-specific? (dev vs prod, OS, browser, Node version)
□ Is it data-specific? (only certain users, certain payloads)
□ Is it timing-specific? (only under load, only after X minutes)
□ When did it first appear? (check git history)
```

### Reproduction Decision Tree
```
Can you reproduce it consistently?
├── YES → Continue to Phase 2
└── NO → Check these possibilities:
    ├── Race condition → See advanced skill
    ├── Stale cache → Force-clear all caches and retest
    ├── Environment drift → Diff the environments
    ├── Data dependency → Test with minimal data set
    └── Timing dependency → Add logging with timestamps
```

### Reproduction Protocol (TypeScript Example)
```typescript
// Step 1: Write a reproduction script FIRST
// This becomes your regression test after the fix

test('reproduces bug #123: user profile fails for names with apostrophes', () => {
  const input = { name: "O'Brien", email: "test@test.com" };
  // This should work but currently throws
  expect(() => createUser(input)).not.toThrow();
  // Or assert on specific output
  const result = createUser(input);
  expect(result.id).toBeDefined();
});

// Step 2: Run the test — it should FAIL
// A failing test proves the bug exists
// A passing test means you haven't reproduced it correctly
```

### Minimal Reproduction Template
```typescript
// Create the smallest possible code that demonstrates the bug
function minimalReproduction() {
  // 1. Set up ONLY the necessary preconditions
  const input = minimalInputThatTriggersBug();
  
  // 2. Execute the buggy code path
  const result = buggyFunction(input);
  
  // 3. Assert the incorrect behavior
  console.log('Expected:', expectedOutput);
  console.log('Actual:', result);
  console.log('Difference:', diff(expectedOutput, result));
}
```

---

## Phase 2: ISOLATE — Narrow to the Smallest Failing Unit

### Binary Search Isolation Method

This is the most efficient way to find bugs in complex systems:

```
1. Identify the full code path from input → output
2. Find the midpoint of that path
3. Add a checkpoint assertion at the midpoint
4. If assertion fails → bug is in first half → recurse
5. If assertion passes → bug is in second half → recurse
6. Continue until you find the exact line
```

### Example: Data Transformation Pipeline
```typescript
// Full path: raw API response → parseResponse() → transformData() → formatForUI() → rendered output

// Step 1: Check at midpoint
const rawData = await fetchRawData();
console.assert(rawData.users.length > 0, 'RAW DATA CHECK: expected users'); // ← checkpoint

const parsed = parseResponse(rawData);
console.assert(parsed.every(u => u.id), 'PARSE CHECK: every user has id'); // ← checkpoint

const transformed = transformData(parsed);
console.assert(transformed.every(u => u.displayName), 'TRANSFORM CHECK'); // ← checkpoint

// Where assertion fails = where bug lives
// Then apply binary search again within that function
```

### Isolation by Simplification
```python
# If complex data triggers a bug, simplify to minimum failing case

# BEFORE (complex, hard to reason about):
data = fetch_from_database(user_id=123, include_relations=True, date_range=(start, end))
result = process(data)  # fails somewhere

# AFTER (isolated, easy to reason about):
minimal_data = {"id": 1, "name": "Test"}  # smallest data that triggers bug
result = process(minimal_data)  # now reproduce with just this
# If this works → bug is in the complex data shape
# If this fails → bug is in process() logic itself
```

### Checkpoint Logging Pattern
```typescript
// Add checkpoints at each transformation step
function debugPipeline(input: unknown) {
  const STEP = 'debugPipeline';
  
  console.log(`[${STEP}] Input:`, JSON.stringify(input, null, 2));
  
  const step1 = parseInput(input);
  console.log(`[${STEP}] After parse:`, step1);
  
  const step2 = validate(step1);
  console.log(`[${STEP}] After validate:`, step2);
  
  const step3 = transform(step2);
  console.log(`[${STEP}] After transform:`, step3);
  
  return step3;
}

// Run with minimal input and trace where it goes wrong
```

---

## Phase 3: CATEGORIZE — Identify the Bug Type

### Bug Type Decision Tree
```
Is the bug deterministic (always fails)?
├── YES → Continue below
└── NO (intermittent) → Go to advanced skill → Race Conditions

Does it fail immediately on first call?
├── YES → Input validation, type error, or missing dependency
└── NO (fails after N calls/time) → State accumulation, memory leak, or resource leak

Is it data-specific?
├── YES → Edge case in data handling (null, empty string, unicode, large numbers)
└── NO → Logic error independent of data

Does it fail only in production/staging but not dev?
├── YES → Environment difference (env vars, network, permissions, data volume)
└── NO → Reproducible bug in logic
```

### Bug Category Reference Table

| Category | Signals | First Suspect | Reference |
|----------|---------|---------------|-----------|
| **Null/Undefined** | TypeError: Cannot read properties of null | Missing null check | Phase 4A |
| **Type Coercion** | Wrong values, works on some inputs | `==` vs `===`, implicit conversion | Phase 4B |
| **Off-by-one** | Last item missing, extra iteration | `<` vs `<=`, 0-indexed vs 1-indexed | Phase 4C |
| **State Mutation** | Works first call, wrong on subsequent | Shared mutable object | Phase 4D |
| **Async Race** | Intermittent, timing-dependent | Promise ordering, missing await | advanced skill |
| **Closure Stale** | React stale state, old values in callback | useEffect deps, event listeners | advanced skill |
| **Memory Leak** | Slows over time, crashes after hours | Subscriptions, timers, event listeners | advanced skill |
| **Logic Error** | Wrong output but no crash | Conditional logic, boundary conditions | Phase 4E |
| **Type Mismatch** | TypeScript works but runtime fails | API returns different shape than expected | Phase 4F |
| **Security** | Injection, auth bypass, data exposure | Input validation, SQL construction | fix skill |
| **Concurrency** | Lost updates, deadlock, corruption | Lock ordering, atomic operations | advanced skill |
| **Performance** | Slow, timeouts, high resource usage | N+1 queries, blocking operations | advanced skill |

---

## Phase 4: ROOT CAUSE PATTERNS

### 4A — Null/Undefined Errors

**SYMPTOM**: "Cannot read properties of undefined (reading 'name')" or similar TypeError

**ROOT CAUSE**: Assuming data exists without verifying

```typescript
// ❌ WRONG: Assumes user always exists
function getUserDisplayName(userId: string): string {
  const user = users.find(u => u.id === userId);
  return user.profile.name; // crashes if user not found OR profile is null
}

// ✅ CORRECT: Guard every assumption
function getUserDisplayName(userId: string): string {
  const user = users.find(u => u.id === userId);
  if (!user) return 'Unknown User';
  if (!user.profile) return user.email ?? 'Unknown User';
  return user.profile.name;
}
```

**DIAGNOSIS CHECKLIST for null errors**:
```
□ Where is this object created/set?
□ Is there a code path where it's never set?
□ Does an async operation populate it? Could it be called before completion?
□ Is it deleted/cleared anywhere before this point?
□ Is there a race between creation and access?
```

---

### 4B — Type Coercion Bugs

**SYMPTOM**: Logic behaves unexpectedly with certain values

**ROOT CAUSE**: JavaScript loose equality or implicit conversion

```typescript
// ❌ These all evaluate TRUE with ==
null == undefined        // true
0 == ''                  // true  
0 == false               // true
'' == false              // true
'0' == false             // true
[] == false              // true

// DIAGNOSIS: Search codebase for `==` (not `===`)
// Fix: Replace every == with === and explicitly handle edge cases

// ❌ WRONG: type coercion causes wrong branch
function processCount(count) {
  if (count == false) {  // catches 0, '', null, undefined, false
    return 'empty';
  }
  return `Count: ${count}`;
}

// ✅ CORRECT: explicit type checking
function processCount(count: number): string {
  if (count === 0) return 'empty';
  if (!Number.isFinite(count)) throw new TypeError(`Expected finite number, got ${count}`);
  return `Count: ${count}`;
}
```

---

### 4C — Off-by-One Errors

**SYMPTOM**: Last element missing, or one extra element, or loop runs one too many times

**ROOT CAUSE**: Incorrect boundary in loop/slice/index

```typescript
// ❌ Common off-by-one patterns:
const arr = [1, 2, 3, 4, 5];

// Missing last element:
for (let i = 0; i < arr.length - 1; i++) { } // should be < arr.length

// Wrong slice (misunderstanding exclusive end):
arr.slice(0, 5) // gives ALL 5 elements (end is exclusive)
arr.slice(0, 4) // gives first 4 elements

// Pagination off-by-one:
const offset = page * pageSize;        // page=1 → offset=10 ✓
const offset = (page - 1) * pageSize; // page=1 → offset=0  ✓ (depends on 0 or 1 indexed)

// ✅ DIAGNOSIS: Write a unit test with known small array
test('pagination returns correct page', () => {
  const items = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
  expect(paginate(items, page=1, size=3)).toEqual([1, 2, 3]);
  expect(paginate(items, page=2, size=3)).toEqual([4, 5, 6]);
  expect(paginate(items, page=4, size=3)).toEqual([10]);      // last partial page
  expect(paginate(items, page=5, size=3)).toEqual([]);        // beyond end
});
```

---

### 4D — State Mutation Bugs

**SYMPTOM**: Works on first call, wrong results on second call

**ROOT CAUSE**: Shared mutable object is modified across calls

```typescript
// ❌ WRONG: defaultOptions is mutated across calls
const defaultOptions = { timeout: 5000, retries: 3, headers: {} };

function makeRequest(url: string, options = defaultOptions) {
  options.headers['Authorization'] = getToken(); // MUTATES shared object!
  return fetch(url, options);
}
// Second call: defaultOptions.headers already has stale Authorization token

// ✅ CORRECT: Always create new objects
const DEFAULT_TIMEOUT = 5000;
const DEFAULT_RETRIES = 3;

function makeRequest(url: string, overrides: Partial<RequestOptions> = {}) {
  const options: RequestOptions = {
    timeout: DEFAULT_TIMEOUT,
    retries: DEFAULT_RETRIES,
    ...overrides,
    headers: {
      ...overrides.headers,
      Authorization: getToken(), // new object, no mutation
    },
  };
  return fetch(url, options);
}
```

**DIAGNOSIS: Look for**:
```
□ Objects passed by reference and modified inside functions
□ Array methods that mutate: push, pop, sort, reverse, splice, fill
□ Default parameter objects
□ Module-level mutable state
□ Closures capturing mutable variables
```

---

### 4E — Logic Errors

**SYMPTOM**: Wrong output, no crash, hard to find

**APPROACH**: Trace each conditional branch with concrete values

```typescript
// ❌ WRONG: Logic error in discount calculation
function calculateDiscount(price: number, memberType: string): number {
  let discount = 0;
  if (memberType === 'gold') discount = 0.2;
  if (memberType === 'silver') discount = 0.1;
  if (price > 100) discount += 0.05; // BUG: applies extra discount even for non-members
  return price * (1 - discount);
}

// DIAGNOSIS: Trace with concrete values
// price=150, memberType='basic'
// discount starts at 0
// gold? no. silver? no.
// price > 100? YES → discount = 0.05 ← wrong! basic members get 5% off

// ✅ CORRECT: Restructure logic to be explicit
function calculateDiscount(price: number, memberType: 'gold' | 'silver' | 'basic'): number {
  const BASE_DISCOUNT: Record<string, number> = {
    gold: 0.20,
    silver: 0.10,
    basic: 0,
  };
  const VOLUME_THRESHOLD = 100;
  const VOLUME_DISCOUNT = 0.05;

  const baseDiscount = BASE_DISCOUNT[memberType] ?? 0;
  const volumeDiscount = (memberType !== 'basic' && price > VOLUME_THRESHOLD)
    ? VOLUME_DISCOUNT
    : 0;

  return price * (1 - baseDiscount - volumeDiscount);
}
```

---

### 4F — API/Type Shape Mismatch

**SYMPTOM**: TypeScript compiles, but runtime crashes with undefined values

**ROOT CAUSE**: Runtime API shape ≠ TypeScript type definition

```typescript
// ❌ WRONG: Trusting type definitions blindly
interface ApiUser {
  id: string;
  name: string;
  email: string;
}

async function getUser(id: string): Promise<ApiUser> {
  const res = await fetch(`/api/users/${id}`);
  return res.json() as ApiUser; // DANGER: No runtime validation
}

// If API returns { id: string, full_name: string, email_address: string }
// TypeScript says it's fine, but user.name and user.email will be undefined at runtime

// ✅ CORRECT: Validate at runtime with Zod
import { z } from 'zod';

const ApiUserSchema = z.object({
  id: z.string(),
  name: z.string(),
  email: z.string().email(),
});
type ApiUser = z.infer<typeof ApiUserSchema>;

async function getUser(id: string): Promise<ApiUser> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(`GET /api/users/${id} failed: ${res.status}`);
  const raw = await res.json();
  return ApiUserSchema.parse(raw); // throws ZodError with clear message if shape is wrong
}
```

---

## Phase 5: DOCUMENT THE ROOT CAUSE

Before writing any fix, write this statement:

```
ROOT CAUSE STATEMENT:
"The bug occurs because [SPECIFIC REASON].
When [INPUT/TRIGGER], the code [WHAT IT DOES WRONG],
but it should [WHAT IT SHOULD DO].
The fix is to [SPECIFIC CHANGE]."
```

### Example Root Cause Statement
```
ROOT CAUSE STATEMENT:
"The bug occurs because `defaultOptions` is a shared module-level object
that gets mutated on every call to makeRequest().
When the function adds the Authorization header, it modifies the shared object,
so subsequent calls inherit the previous call's headers.
The fix is to create a new options object on each call using spread syntax."
```

**If you cannot write this statement clearly, you do not understand the bug yet. Do not write a fix until you can.**

---

## 🔷 RAPID DIAGNOSIS CHECKLIST

For quick diagnosis, run through this checklist:

```
□ Error message captured and analyzed
□ Stack trace examined for exact failure location
□ Input that triggers bug identified
□ Expected vs actual behavior documented
□ Code path from input to failure traced
□ All branches in code path tested with values
□ Related tests examined for expectations
□ Git history checked for recent changes in area
□ Environment differences ruled out
□ Data dependency identified or ruled out
```

---

## 🔷 DEBUGGING TOOLS BY LANGUAGE

### JavaScript/TypeScript
```bash
# Node.js debugging
node --inspect-brk script.js

# Chrome DevTools
# Open chrome://inspect

# Console debugging
console.trace('label');
console.table(array);
console.time('label'); /* code */ console.timeEnd('label');
```

### Python
```python
# pdb debugger
import pdb; pdb.set_trace()

# or from command line
python -m pdb script.py

# pprint for complex data
from pprint import pprint
pprint(complex_object)

# logging with traceback
import logging
logging.basicConfig(level=logging.DEBUG)
try:
    risky_operation()
except Exception:
    logging.exception("Failed:")
```

### React
```tsx
// React DevTools - browser extension
// Check component props and state

// useEffect debugging
useEffect(() => {
  console.log('Effect ran with:', { dep1, dep2 });
  return () => console.log('Effect cleanup');
}, [dep1, dep2]);

// Render debugging
function Component(props) {
  console.log('Rendering with props:', props);
  // ...
}
```

---

## 🔷 VIP EXPERT NOTE: DIAGNOSIS ACCELERATION TECHNIQUES

### Technique 1: The "Last Known Good" Method (Git Bisect)
```
1. Find the last commit/deploy where this worked
2. Read the diff between "last good" and "first bad"
3. The bug is in that diff — guaranteed
4. This reduces search space from entire codebase to a few changed files
```

```bash
# Git bisect — binary search through commits to find the breaking change
git bisect start
git bisect bad          # current commit is broken
git bisect good abc123  # this commit was working
# Git checks out middle commit — test it, then run:
git bisect good  # or  git bisect bad
# Repeat until Git tells you which commit introduced the bug
git bisect reset  # when done
```

### Technique 2: The "Explain to a Rubber Duck" Method
```
Before writing any fix, explain the bug out loud (or in writing):
1. "The function receives [X] as input"
2. "At step [N], it does [Y]"
3. "But it should do [Z] because [REASON]"
4. "The discrepancy is because [ROOT CAUSE]"

If you can't complete step 4, you don't understand the bug yet.
```

### Technique 3: The "What Changed?" Method
```
For bugs that suddenly appear:
□ What code was deployed recently? (check deploy log)
□ What dependencies were updated? (check package-lock diff)
□ What environment changed? (env vars, infrastructure, DNS)
□ What data changed? (new user inputs, edge case data)
□ What time-dependent code exists? (timezones, expiry, cron)
```

---

## 🔷 WHEN TO ESCALATE

Escalate to advanced skill when:

- Bug is intermittent (race condition)
- Bug involves timing/deadlines (async)
- Bug appears after hours of runtime (memory leak)
- Bug only under load (concurrency)
- Bug involves multiple processes/threads
- Bug involves database transactions
- Bug involves distributed systems

---

*This skill provides systematic diagnosis methodology. Always complete diagnosis before attempting fixes.*
