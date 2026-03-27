# 🧪 SKILL: Test Strategy — High-Value, High-Validity Tests

## 🎯 Purpose

Write tests that actually catch bugs — not tests that just increase coverage numbers. A test that doesn't fail when the code is wrong is worse than no test.

> **VIP EXPERT NOTE — Test ROI Pyramid:**
> Not all tests have equal value. Prioritize by return on investment:
> ```
> HIGHEST ROI:  Regression tests (prevent known bugs from returning)
>               Edge case tests (null, empty, boundary, overflow)
>               Error path tests (what happens when things fail)
>               Integration tests at I/O boundaries
> LOWEST ROI:   Snapshot tests (brittle, low signal)
>               Tests that mirror implementation (break on refactor)
>               Tests with no assertions (false coverage)
> ```

---

## 🔷 THE TEST HIERARCHY (Write in This Order)

```
1. REGRESSION TEST  → Fails on the current bug (proves bug exists)
2. UNIT TEST        → Tests the isolated function/module
3. INTEGRATION TEST → Tests the flow end-to-end
4. EDGE CASE TESTS  → Null, empty, boundary, overflow, unicode, concurrent
```

**Never start with integration tests. Never skip the regression test for bugs.**

---

## 🔷 TEST QUALITY STANDARDS

### ✅ A Good Test

```typescript
// GOOD: Descriptive name, tests behavior, one clear assertion per concept
describe('calculateDiscount', () => {
  it('applies 20% discount for gold members', () => {
    const result = calculateDiscount({ price: 100, memberType: 'gold' });
    expect(result).toBe(80);
  });

  it('applies no discount for basic members regardless of price', () => {
    const result = calculateDiscount({ price: 500, memberType: 'basic' });
    expect(result).toBe(500); // no discount
  });

  it('throws TypeError when price is negative', () => {
    expect(() => calculateDiscount({ price: -10, memberType: 'gold' }))
      .toThrow(TypeError);
  });
});
```

### ❌ Bad Tests (Anti-Patterns)

```typescript
// BAD: Tests implementation, not behavior. Will break on any refactor.
it('test1', () => {
  const d = cd(100, 'g'); // cryptic abbreviations
  expect(d).toBe(80);     // what does 80 mean? no context
});

// BAD: Multiple unrelated concepts in one test
it('calculates discount', () => {
  expect(calculateDiscount(100, 'gold')).toBe(80);
  expect(calculateDiscount(100, 'silver')).toBe(90);
  expect(calculateDiscount(0, 'gold')).toBe(0);
  expect(calculateDiscount(-10, 'gold')).toThrow();
  // When this fails, you don't know WHICH case failed without running it
});

// BAD: Logic inside test — tests can have bugs too
it('applies correct discount', () => {
  const discount = memberType === 'gold' ? 0.2 : 0.1; // logic in test!
  expect(calculateDiscount(100, memberType)).toBe(100 * (1 - discount));
});
```

---

## 🔷 THE ARRANGE-ACT-ASSERT PATTERN (Mandatory)

Every test follows this structure:

```typescript
it('description of expected behavior', () => {
  // ARRANGE — set up test data and preconditions
  const user = createTestUser({ role: 'admin', isActive: true });
  const resource = createTestResource({ ownerId: 'other-user' });

  // ACT — execute exactly one action
  const result = canUserAccessResource(user, resource);

  // ASSERT — verify the outcome
  expect(result).toBe(true); // admins can access any resource
});
```

**Separation rule**: If you can't clearly label each section, your test is doing too much.

### Test Naming Formula (VIP EXPERT NOTE)

```
Good test names follow this formula:
  "[unit] [behavior] [condition]"

✅ 'calculateDiscount applies 20% discount for gold members'
✅ 'fetchUser returns null when user not found (404)'
✅ 'validateEmail rejects email without @ symbol'
✅ 'processOrder throws InsufficientFundsError when balance < total'

❌ 'test1'
❌ 'it works'
❌ 'should be correct'
❌ 'calculateDiscount test'

Advanced formula for regression tests:
  "reproduces bug #[ID]: [exact scenario]"

✅ 'reproduces bug #456: double-click creates duplicate orders'
✅ 'reproduces bug #789: Unicode names break profile display'
```

---

## 🔷 COMPREHENSIVE TEST MATRIX

For each function/method, systematically verify these cases:

### Numeric Inputs
```typescript
describe('numeric edge cases', () => {
  it('handles zero', () => expect(fn(0)).toBeDefined());
  it('handles negative numbers', () => expect(() => fn(-1)).toThrow());
  it('handles very large numbers', () => expect(fn(Number.MAX_SAFE_INTEGER)).toBeDefined());
  it('handles decimals', () => expect(fn(0.1 + 0.2)).toBeDefined()); // floating point!
  it('handles NaN', () => expect(() => fn(NaN)).toThrow(TypeError));
  it('handles Infinity', () => expect(() => fn(Infinity)).toThrow(TypeError));
});
```

### String Inputs
```typescript
describe('string edge cases', () => {
  it('handles empty string', () => expect(fn('')).toBeDefined());
  it('handles whitespace-only string', () => expect(fn('   ')).toBeDefined());
  it('handles very long string', () => expect(fn('a'.repeat(10000))).toBeDefined());
  it('handles special characters', () => expect(fn('O\'Brien & <Test>')).toBeDefined());
  it('handles unicode', () => expect(fn('日本語テスト')).toBeDefined());
  it('handles null (if typed as string | null)', () => expect(fn(null)).toBeDefined());
});
```

### Array/Collection Inputs
```typescript
describe('collection edge cases', () => {
  it('handles empty array', () => expect(fn([])).toEqual([]));
  it('handles single element', () => expect(fn([1])).toBeDefined());
  it('handles array with null elements', () => expect(fn([null, 1, null])).toBeDefined());
  it('handles very large array', () => {
    const large = Array.from({ length: 100000 }, (_, i) => i);
    expect(() => fn(large)).not.toThrow();
  });
  it('handles duplicates', () => expect(fn([1, 1, 1])).toBeDefined());
});
```

### Async Functions
```typescript
describe('async edge cases', () => {
  it('resolves successfully', async () => {
    await expect(asyncFn()).resolves.toBeDefined();
  });

  it('rejects with correct error type on failure', async () => {
    mockApi.mockRejectedValueOnce(new NetworkError('timeout'));
    await expect(asyncFn()).rejects.toThrow(NetworkError);
  });

  it('handles concurrent calls safely', async () => {
    const results = await Promise.all([asyncFn('a'), asyncFn('b'), asyncFn('c')]);
    expect(results).toHaveLength(3);
    expect(results.every(r => r !== undefined)).toBe(true);
  });

  it('respects cancellation / cleanup', async () => {
    const controller = new AbortController();
    const promise = asyncFn({ signal: controller.signal });
    controller.abort();
    await expect(promise).rejects.toThrow('aborted');
  });
});
```

---

## 🔷 TEST DATA PATTERNS

### Use Factories, Not Hardcoded Objects

```typescript
// ❌ WRONG: Hardcoded test data breaks on schema changes
const user = {
  id: '123',
  name: 'Test User',
  email: 'test@example.com',
  role: 'admin',
  createdAt: new Date('2024-01-01'),
  // adding a required field here breaks 50 tests
};

// ✅ CORRECT: Factory with sensible defaults
function createTestUser(overrides: Partial<User> = {}): User {
  return {
    id: overrides.id ?? crypto.randomUUID(),
    name: overrides.name ?? 'Test User',
    email: overrides.email ?? `test-${Date.now()}@example.com`,
    role: overrides.role ?? 'viewer',
    isActive: overrides.isActive ?? true,
    createdAt: overrides.createdAt ?? new Date(),
    ...overrides,
  };
}

// Usage: only specify what matters for the test
const admin = createTestUser({ role: 'admin' });
const inactiveUser = createTestUser({ isActive: false });
```

### Use Realistic Data

```typescript
// ❌ WRONG: "foo", "bar", "test" hide real-world bugs
const user = { name: 'foo', email: 'bar' };

// ✅ CORRECT: Realistic data catches real-world edge cases
const user = { name: "O'Brien", email: 'alice.o-brien+tag@company.co.uk' };
// This catches:
// - SQL injection via apostrophe
// - Email regex that fails on + or -
// - Domain parsing bugs with .co.uk
```

---

## 🔷 MOCKING STRATEGY

### Mock at the Boundary, Not Internally

```typescript
// ❌ WRONG: Mocking internal implementation (brittle)
jest.mock('./utils/formatDate'); // mocking a utility used internally

// ✅ CORRECT: Mock at the I/O boundary
jest.mock('./api/client'); // mock the HTTP client
jest.mock('./db/connection'); // mock the database connection

// Internal logic runs real code — only I/O is mocked
```

### Mock Return Values Precisely

```typescript
// ❌ WRONG: Mock returns anything — doesn't test integration with real shape
mockFetch.mockResolvedValue({ data: { id: '1' } });

// ✅ CORRECT: Mock returns exact production-like shape
mockFetch.mockResolvedValue({
  ok: true,
  status: 200,
  json: async () => ({
    id: '550e8400-e29b-41d4-a716-446655440000',
    name: 'Alice Johnson',
    email: 'alice@example.com',
    role: 'admin' as const,
    createdAt: '2024-01-15T10:30:00.000Z',
  }),
});
```

### Always Test the Error Path

```typescript
// For every happy path test, write a corresponding error path test
describe('fetchUser', () => {
  it('returns user when found', async () => {
    mockApi.mockResolvedValueOnce(createMockUser());
    const result = await fetchUser('123');
    expect(result).toMatchObject({ id: '123' });
  });

  it('returns null when user not found (404)', async () => {
    mockApi.mockRejectedValueOnce(new ApiError(404));
    const result = await fetchUser('nonexistent');
    expect(result).toBeNull();
  });

  it('throws when server errors (500)', async () => {
    mockApi.mockRejectedValueOnce(new ApiError(500));
    await expect(fetchUser('123')).rejects.toThrow('fetchUser failed');
  });

  it('throws TypeError for invalid id type', async () => {
    await expect(fetchUser('')).rejects.toThrow(TypeError);
    await expect(fetchUser(null as any)).rejects.toThrow(TypeError);
  });
});
```

---

## 🔷 INTEGRATION TEST TEMPLATE

```typescript
// Integration test: tests the full flow with minimal mocking
describe('User Registration Flow (Integration)', () => {
  let db: TestDatabase;
  let emailService: MockEmailService;

  beforeEach(async () => {
    db = await TestDatabase.create(); // isolated test DB
    emailService = new MockEmailService();
  });

  afterEach(async () => {
    await db.destroy(); // clean up
  });

  it('creates user, sends welcome email, and returns user with id', async () => {
    // ARRANGE
    const input = {
      name: 'Alice Johnson',
      email: 'alice@example.com',
      password: 'SecurePass123!',
    };

    // ACT
    const result = await registerUser(input, { db, emailService });

    // ASSERT — verify all side effects
    expect(result.id).toMatch(/^[0-9a-f-]{36}$/); // UUID format
    expect(result.email).toBe(input.email);
    expect(result.passwordHash).not.toBe(input.password); // password was hashed

    const dbUser = await db.users.findById(result.id);
    expect(dbUser).not.toBeNull(); // persisted to DB

    expect(emailService.sent).toHaveLength(1);
    expect(emailService.sent[0]).toMatchObject({
      to: input.email,
      subject: expect.stringContaining('Welcome'),
    });
  });

  it('rejects duplicate email with DuplicateEmailError', async () => {
    await registerUser({ email: 'alice@example.com', ...defaults }, { db, emailService });

    await expect(
      registerUser({ email: 'alice@example.com', ...defaults }, { db, emailService })
    ).rejects.toThrow(DuplicateEmailError);

    // Verify only one user in DB
    const users = await db.users.findByEmail('alice@example.com');
    expect(users).toHaveLength(1);
  });
});
```

---

## 🔷 COVERAGE QUALITY GUIDELINES

### Coverage Numbers Are Misleading

```
80% coverage means 20% is untested.
But even 100% coverage doesn't mean tests are good.
A test that calls every line but never asserts anything = 100% coverage, 0% value.
```

### Coverage That MATTERS

```
✅ Every error path is tested (not just happy path)
✅ Every guard clause is tested (what happens when it triggers)
✅ Every boundary condition is tested (0, 1, max, max+1)
✅ Every external I/O is mocked and error cases are tested
✅ State machine transitions are all tested
```

### Mutation Testing (Advanced)

```bash
# Run mutation testing to verify your tests actually catch bugs
npx stryker run
# Stryker changes your code (flips conditions, removes returns)
# If your tests still pass = they weren't actually testing the logic
# Good tests should fail on ~80%+ of mutations
```

---

## 🔷 TEST TYPES BY PURPOSE

### Unit Tests
- **Purpose**: Test isolated logic
- **Speed**: Fast (< 10ms each)
- **Scope**: Single function/method
- **Mocking**: Mock all dependencies

### Integration Tests
- **Purpose**: Test component interaction
- **Speed**: Medium (10ms - 1s)
- **Scope**: Multiple components
- **Mocking**: Mock external services only

### End-to-End Tests
- **Purpose**: Test user flows
- **Speed**: Slow (1s - 30s)
- **Scope**: Full application
- **Mocking**: None or minimal

### Regression Tests
- **Purpose**: Prevent specific bugs from returning
- **Format**: Exact reproduction of bug scenario
- **Priority**: Always write for every fixed bug

### Property-Based Tests
- **Purpose**: Find edge cases you didn't think of
- **Tool**: fast-check, jsverify, hypothesis

```typescript
// Property-based testing example
import fc from 'fast-check';

test('sort is idempotent', () => {
  fc.assert(
    fc.property(fc.array(fc.integer()), (arr) => {
      const sorted = sort(arr);
      expect(sort(sorted)).toEqual(sorted); // sorting twice = sorting once
    })
  );
});

test('sort contains same elements', () => {
  fc.assert(
    fc.property(fc.array(fc.integer()), (arr) => {
      const sorted = sort(arr);
      expect(sorted.length).toBe(arr.length);
      expect(sorted.sort()).toEqual([...arr].sort());
    })
  );
});
```

---

## 🔷 VIP EXPERT NOTE: CONTRACT TESTING

> **When to use:** Any time your code talks to an external API, database, or service.
> Contract tests verify the *shape* of data at boundaries — they catch the #1 source
> of production bugs: "the API changed and nobody told us."

```typescript
// Contract test: verify API response shape matches our types
import { z } from 'zod';

const UserApiContract = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  email: z.string().email(),
  role: z.enum(['admin', 'editor', 'viewer']),
  createdAt: z.string().datetime(),
});

describe('User API Contract', () => {
  it('GET /api/users/:id returns data matching UserApiContract', async () => {
    const response = await fetch('/api/users/test-user-id');
    const data = await response.json();

    // This will throw with a detailed error if the API shape changed
    const result = UserApiContract.safeParse(data);
    if (!result.success) {
      throw new Error(
        `API contract violation:\n${result.error.issues.map(i => `  ${i.path}: ${i.message}`).join('\n')}`
      );
    }
  });
});
```

---

## 🔷 VIP EXPERT NOTE: SNAPSHOT TESTING (Use With Caution)

```typescript
// ⚠️ CAUTION: Snapshot tests are LOW-VALUE tests in most cases.
// They break on any change (even cosmetic), creating noise.
// Use snapshots ONLY for:
//   1. Serialized data structures (JSON API responses)
//   2. Error messages that must remain stable
//   3. Configuration objects

// ❌ BAD USE: Snapshot of rendered component (breaks on any CSS/HTML change)
it('renders correctly', () => {
  const tree = render(<UserProfile user={mockUser} />);
  expect(tree).toMatchSnapshot(); // Breaks on every visual tweak
});

// ✅ GOOD USE: Snapshot of API response shape
it('returns expected response shape', async () => {
  const response = await createUser(validInput);
  expect(response).toMatchInlineSnapshot(`
    {
      "id": Any<String>,
      "email": "test@example.com",
      "role": "viewer",
    }
  `);
});
```

---

## 🔷 VIP EXPERT NOTE: TEST SMELL DETECTION

```
Smell                          | Problem                           | Fix
-------------------------------|-----------------------------------|---------------------------
Test relies on execution order | Shared state between tests        | Use beforeEach for setup
Test uses setTimeout/sleep     | Timing-dependent = flaky          | Use waitFor/polling
Test has >5 assertions         | Testing too many things at once   | Split into separate tests
Test name is generic            | Can't tell what broke from name   | Use naming formula above
Test mocks everything          | Not testing real behavior          | Mock only I/O boundaries
Test requires specific DB data | Fragile, breaks on data changes   | Use factories, seed in test
Test passes when code is wrong | False positive — no real assertion | Add meaningful assertions
```

---

## 🔷 REACT TESTING PATTERNS

### Component Testing Template

```tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { UserProfile } from './UserProfile';

describe('UserProfile', () => {
  const mockUser = createTestUser();

  it('displays user information when loaded', async () => {
    render(<UserProfile userId={mockUser.id} />);

    await waitFor(() => {
      expect(screen.getByText(mockUser.name)).toBeInTheDocument();
    });
  });

  it('shows loading state initially', () => {
    render(<UserProfile userId={mockUser.id} />);
    expect(screen.getByRole('progressbar')).toBeInTheDocument();
  });

  it('shows error message when fetch fails', async () => {
    mockFetchUser.mockRejectedValueOnce(new Error('Network error'));
    render(<UserProfile userId={mockUser.id} />);

    await waitFor(() => {
      expect(screen.getByText(/error/i)).toBeInTheDocument();
    });
  });

  it('calls onEdit when edit button clicked', async () => {
    const onEdit = jest.fn();
    render(<UserProfile userId={mockUser.id} onEdit={onEdit} />);

    await waitFor(() => screen.getByText(mockUser.name));
    await userEvent.click(screen.getByRole('button', { name: /edit/i }));

    expect(onEdit).toHaveBeenCalledWith(mockUser.id);
  });
});
```

### Hook Testing

```typescript
import { renderHook, waitFor } from '@testing-library/react';
import { useUserData } from './useUserData';

describe('useUserData', () => {
  it('returns loading state initially', () => {
    const { result } = renderHook(() => useUserData('123'));
    expect(result.current.status).toBe('loading');
  });

  it('returns user data on success', async () => {
    const mockUser = createTestUser();
    mockFetchUser.mockResolvedValueOnce(mockUser);

    const { result } = renderHook(() => useUserData('123'));

    await waitFor(() => {
      expect(result.current.status).toBe('success');
    });
    expect(result.current.data).toEqual(mockUser);
  });
});
```

---

## 🔷 PYTHON TESTING PATTERNS

### pytest Best Practices

```python
import pytest
from datetime import datetime
from myapp.users import create_user, UserService

# Fixtures for test data
@pytest.fixture
def user_service(db_session):
    return UserService(db_session)

@pytest.fixture
def sample_user_data():
    return {
        "name": "Alice Johnson",
        "email": "alice@example.com",
        "password": "SecurePass123!",
    }

# Parameterized tests for edge cases
@pytest.mark.parametrize("email,expected_valid", [
    ("test@example.com", True),
    ("invalid-email", False),
    ("@nodomain.com", False),
    ("no@domain.", False),
    ("unicode@例え.jp", True),
])
def test_email_validation(email, expected_valid):
    assert is_valid_email(email) == expected_valid

# Testing exceptions
def test_create_user_duplicate_email(user_service, sample_user_data):
    user_service.create(sample_user_data)
    
    with pytest.raises(DuplicateEmailError) as exc_info:
        user_service.create(sample_user_data)
    
    assert "already exists" in str(exc_info.value)

# Testing with mocks
def test_send_welcome_email_called(mocker, user_service, sample_user_data):
    mock_send = mocker.patch('myapp.email.send_email')
    
    user_service.create(sample_user_data)
    
    mock_send.assert_called_once()
    call_kwargs = mock_send.call_args.kwargs
    assert call_kwargs['to'] == sample_user_data['email']
    assert 'Welcome' in call_kwargs['subject']
```

---

## 🔷 TEST QUALITY CHECKLIST

Before considering any test suite complete, verify:

```
□ Test name describes BEHAVIOR not implementation
□ One assertion per concept (multiple related assertions OK)
□ Uses realistic data, not "foo"/"bar"
□ Tests failure path, not just happy path
□ No logic inside tests (no if/for)
□ Tests are deterministic (no random, no time-dependent without mocking)
□ Tests are isolated (no shared state between tests)
□ Tests have clear failure messages
□ Error paths are tested with specific error types
□ Edge cases are covered (null, empty, boundaries)
□ Integration points are mocked at boundaries
□ Tests clean up after themselves
```

---

## 🔷 VIP EXPERT NOTE: THE "MUTANT KILL" RULE

> **The ultimate test quality metric:** If you change (mutate) one line of your
> production code — flip a condition, remove a return, change a + to a - — at
> least one test MUST fail. If no test fails, your tests are *decorative*, not protective.

```bash
# Run mutation testing to prove your tests actually work:
# JavaScript/TypeScript:
npx stryker run

# Python:
pip install mutmut && mutmut run

# Target: Kill rate ≥ 80% of mutations
# If lower, your tests need more assertions on real behavior
```

---

*This skill provides comprehensive test strategy. Always write regression tests for bugs before fixing them.*
