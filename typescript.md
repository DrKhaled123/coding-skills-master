# TypeScript Patterns — Type Safety & Professional Development Guide

> **VIP EXPERT NOTE:** TypeScript's type system is your most powerful bug prevention
> tool. 60% of runtime crashes in JavaScript apps are type-related errors that
> TypeScript catches at compile time — *if you use it correctly*. The patterns below
> show how to extract maximum safety from the type system and write maintainable,
> self-documenting code.

---

## 🔷 DISCRIMINATED UNIONS (Most Powerful TypeScript Pattern)

```typescript
// ✅ GOLD STANDARD: Discriminated union — TypeScript narrows types automatically
type ApiResponse<T> =
  | { status: 'success'; data: T; requestId: string }
  | { status: 'error'; code: string; message: string; requestId: string }
  | { status: 'loading' };

function renderResponse<T>(response: ApiResponse<T>, render: (data: T) => React.ReactNode) {
  switch (response.status) {
    case 'loading':
      return <Spinner />;
    case 'error':
      return <ErrorBanner code={response.code} message={response.message} />;
      // TypeScript knows: response.data does NOT exist here
    case 'success':
      return render(response.data);
      // TypeScript knows: response.data is T here — no casting needed
  }
}

// Domain modeling with discriminated unions:
type PaymentMethod =
  | { type: 'credit_card'; last4: string; expiry: string; brand: 'visa' | 'mastercard' | 'amex' }
  | { type: 'paypal'; email: string }
  | { type: 'bank_transfer'; accountLast4: string; bankName: string }
  | { type: 'crypto'; address: string; currency: 'BTC' | 'ETH' | 'USDC' };

function getPaymentDisplayName(method: PaymentMethod): string {
  switch (method.type) {
    case 'credit_card': return `${method.brand} ****${method.last4}`;
    case 'paypal': return `PayPal (${method.email})`;
    case 'bank_transfer': return `${method.bankName} ****${method.accountLast4}`;
    case 'crypto': return `${method.currency} (${method.address.slice(0, 6)}...)`;
    default: {
      const _exhaustive: never = method;
      throw new Error(`Unknown payment method: ${JSON.stringify(_exhaustive)}`);
    }
  }
}
```

> **VIP EXPERT NOTE:** Discriminated unions are TypeScript's most powerful feature.
> They eliminate impossible states, provide automatic type narrowing, and make your
> code self-documenting. The `never` exhaustiveness check ensures you handle all
> cases — if you add a new payment method, TypeScript will error until you update
> the switch statement.

---

## 🔷 UTILITY TYPES (Use, Don't Reinvent)

```typescript
// ✅ Use built-in utility types instead of duplicating types

interface User {
  id: string;
  name: string;
  email: string;
  passwordHash: string;
  role: 'admin' | 'editor' | 'viewer';
  createdAt: Date;
  updatedAt: Date;
}

// For API responses (exclude sensitive fields):
type PublicUser = Omit<User, 'passwordHash'>;

// For update requests (all fields optional except id):
type UpdateUserInput = Partial<Omit<User, 'id' | 'createdAt' | 'updatedAt'>>;

// For create requests (no auto-generated fields):
type CreateUserInput = Omit<User, 'id' | 'createdAt' | 'updatedAt'>;

// Read-only version (prevents mutation):
type ImmutableUser = Readonly<User>;

// Pick specific fields:
type UserCredentials = Pick<User, 'email' | 'passwordHash'>;

// Record type for lookups:
type UserById = Record<string, User>;

// Extract data from generic types:
type ApiResponseData<T> = T extends ApiResponse<infer D> ? D : never;

// Deep partial for nested updates
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

// Non-nullable version
type RequiredUser = Required<Pick<User, 'name' | 'email'>>;

// VIP NOTE: These are all BUILT-IN utility types. Don't reinvent them!
// TypeScript provides: Partial, Required, Readonly, Pick, Omit, Record,
// Exclude, Extract, NonNullable, ReturnType, Parameters, InstanceType,
// Awaited, and more. Check the docs before creating your own.
```

> **VIP EXPERT NOTE:** TypeScript's utility types eliminate duplication and keep
> types in sync. Learn them by heart: `Pick` for subtypes, `Omit` for exclusions,
> `Partial` for updates, `Required` for guarantees. Create your own mapped types
> (like `DeepPartial`) for domain-specific transformations.

---

## 🔷 GENERIC CONSTRAINTS

```typescript
// ❌ BROKEN: Too loose — accepts anything
function getProperty(obj: any, key: string) {
  return obj[key]; // returns any
}

// ✅ CORRECT: Constrained generic — type-safe property access
function getProperty<T extends object, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: '1', name: 'Alice', age: 30 };
const name = getProperty(user, 'name');   // TypeScript: string
const age = getProperty(user, 'age');     // TypeScript: number
const bad = getProperty(user, 'email');  // TypeScript ERROR: 'email' not in User

// Generic repository pattern:
interface Repository<T extends { id: string }> {
  findById(id: string): Promise<T | null>;
  findMany(filter: Partial<T>): Promise<T[]>;
  create(data: Omit<T, 'id'>): Promise<T>;
  update(id: string, data: Partial<Omit<T, 'id'>>): Promise<T>;
  delete(id: string): Promise<void>;
}

class UserRepository implements Repository<User> {
  async findById(id: string): Promise<User | null> {
    return db.users.findUnique({ where: { id } });
  }
  // ... fully typed implementation
}

// Multiple constraints
function merge<T extends object, U extends object>(obj1: T, obj2: U): T & U {
  return { ...obj1, ...obj2 };
}

// Generic with default
type ApiResponse<T = unknown> = {
  data: T;
  status: number;
};
```

> **VIP EXPERT NOTE:** Generic constraints (`extends`) give you the flexibility of
> generics with the safety of specific types. Use `keyof` constraints for property
> access patterns. Use multiple generics for independent type parameters. The
> repository pattern example shows how generics enable reusable, type-safe data access.

---

## 🔷 MAPPED TYPES FOR VALIDATION

```typescript
// ✅ Generate validation schemas from types automatically
type ValidationRule<T> = {
  required?: boolean;
  validator?: (value: T) => string | null; // null = valid, string = error message
};

// Mapped type: same keys as T, but each value is a ValidationRule
type ValidationSchema<T> = {
  [K in keyof T]?: ValidationRule<T[K]>;
};

function validate<T extends object>(
  data: T,
  schema: ValidationSchema<T>
): Record<keyof T, string | null> {
  const errors = {} as Record<keyof T, string | null>;

  for (const key in schema) {
    const rule = schema[key];
    const value = data[key];

    if (rule?.required && (value === null || value === undefined || value === '')) {
      errors[key] = `${String(key)} is required`;
      continue;
    }

    if (rule?.validator && value !== null && value !== undefined) {
      errors[key] = rule.validator(value);
    } else {
      errors[key] = null;
    }
  }

  return errors;
}

// Usage — fully typed, no magic strings
const schema: ValidationSchema<CreateUserInput> = {
  name: { required: true, validator: (v) => v.length > 100 ? 'Name too long' : null },
  email: { required: true, validator: (v) => v.includes('@') ? null : 'Invalid email' },
  password: { required: true, validator: (v) => v.length >= 8 ? null : 'Too short' },
};
```

> **VIP EXPERT NOTE:** Mapped types (`{ [K in keyof T]: ... }`) transform existing
> types into new shapes. This validation example maps data fields to their validation
> rules, keeping the schema in sync with the type definition. Add `readonly` or `?`
> modifiers to change property attributes in the mapped type.

---

## 🔷 TEMPLATE LITERAL TYPES

```typescript
// ✅ Type-safe string patterns
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
type ApiVersion = 'v1' | 'v2';
type ApiEndpoint = `/${ApiVersion}/${string}`;

// Typed event names:
type UserEvent = `user:${'created' | 'updated' | 'deleted' | 'login' | 'logout'}`;
type OrderEvent = `order:${'placed' | 'shipped' | 'delivered' | 'cancelled'}`;
type AppEvent = UserEvent | OrderEvent;

function emit(event: AppEvent, payload: unknown): void {
  eventBus.emit(event, payload);
}

emit('user:created', { id: '1' });  // ✅ valid
emit('user:banned', { id: '1' });   // ❌ TypeScript ERROR

// CSS class name type safety:
type Breakpoint = 'sm' | 'md' | 'lg' | 'xl';
type SpacingValue = 1 | 2 | 4 | 8 | 16;
type TailwindPadding = `p-${SpacingValue}` | `p${Breakpoint}:p-${SpacingValue}`;

// File path patterns
type ImageExtension = 'png' | 'jpg' | 'jpeg' | 'gif' | 'webp';
type ImagePath = `/images/${string}.${ImageExtension}`;

// Environment variable patterns
type EnvVar = `REACT_APP_${string}` | `NEXT_PUBLIC_${string}`;
```

> **VIP EXPERT NOTE:** Template literal types bring type safety to string patterns.
> Use them for: API routes, event names, CSS classes, file paths, and environment
> variables. Combine with unions for exhaustive pattern matching. The `string`
> template literal type captures any string in that position.

---

## 🔷 TYPE GUARDS AND NARROWING

```typescript
// ✅ Custom type guards for safe narrowing
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value && typeof (value as any).id === 'string' &&
    'email' in value && typeof (value as any).email === 'string' &&
    'role' in value && ['admin', 'editor', 'viewer'].includes((value as any).role)
  );
}

// Use in API response parsing
async function fetchUser(id: string): Promise<User> {
  const response = await api.get(`/users/${id}`);
  const data = await response.json();

  if (!isUser(data)) {
    throw new TypeError(`API returned invalid user shape: ${JSON.stringify(data)}`);
  }

  return data; // TypeScript knows this is User
}

// Assertion function (throws instead of returning bool):
function assertIsUser(value: unknown): asserts value is User {
  if (!isUser(value)) {
    throw new TypeError(`Expected User, got: ${JSON.stringify(value)}`);
  }
}

// Usage
function processUser(data: unknown) {
  assertIsUser(data);
  // TypeScript knows data is User here
  console.log(data.email); // ✅ Safe
}

// typeof guards
function processValue(value: string | number | boolean) {
  if (typeof value === 'string') {
    return value.toUpperCase(); // TypeScript knows: string
  }
  if (typeof value === 'number') {
    return value.toFixed(2); // TypeScript knows: number
  }
  return value ? 'yes' : 'no'; // TypeScript knows: boolean
}

// instanceof guards
function handleError(error: Error | NetworkError | ValidationError) {
  if (error instanceof NetworkError) {
    return retryRequest(); // TypeScript knows: NetworkError
  }
  if (error instanceof ValidationError) {
    return showValidation(error.fields); // TypeScript knows: ValidationError
  }
  throw error; // TypeScript knows: Error
}

// in operator guards
type Cat = { meow: () => void };
type Dog = { bark: () => void };

function makeSound(animal: Cat | Dog) {
  if ('meow' in animal) {
    animal.meow(); // TypeScript knows: Cat
  } else {
    animal.bark(); // TypeScript knows: Dog
  }
}
```

> **VIP EXPERT NOTE:** Type guards are user-defined type checks. The `value is Type`
> return type tells TypeScript to narrow the type when the function returns true.
> Assertion functions (`asserts value is Type`) narrow after the call. Use typeof,
> instanceof, and `in` for built-in narrowing patterns.

---

## 🔷 BRANDED TYPES FOR TYPE-SAFE IDs

```typescript
// ✅ Prevent mixing up IDs of different types
type Brand<T, B> = T & { __brand: B };

type UserId = Brand<string, 'UserId'>;
type OrderId = Brand<string, 'OrderId'>;
type ProductId = Brand<string, 'ProductId'>;

// Create branded IDs
function createUserId(id: string): UserId {
  return id as UserId;
}

function createOrderId(id: string): OrderId {
  return id as OrderId;
}

// Now TypeScript prevents mixing
function getUser(id: UserId): User { /* ... */ }
function getOrder(id: OrderId): Order { /* ... */ }

const userId = createUserId('user-123');
const orderId = createOrderId('order-456');

getUser(userId);   // ✅ OK
getUser(orderId);  // ❌ TypeScript ERROR: OrderId not assignable to UserId

// Branded types for units
type Meters = Brand<number, 'Meters'>;
type Seconds = Brand<number, 'Seconds'>;

function meters(value: number): Meters {
  return value as Meters;
}

function seconds(value: number): Seconds {
  return value as Seconds;
}

const distance = meters(100);
const time = seconds(10);
const speed = distance / time; // TypeScript allows: number
```

> **VIP EXPERT NOTE:** Branded types use intersection types and phantom fields to
> create distinct types at compile time that don't exist at runtime. They're
> essential for type-safe IDs (preventing `userId` vs `orderId` mixups) and
> unit types (meters vs seconds). The brand field `__brand` never actually exists
> — it's erased during compilation.

---

## 🔷 CONST ASSERTIONS

```typescript
// ✅ Lock down literal types with const assertion
const ROLES = ['admin', 'editor', 'viewer'] as const;
type Role = typeof ROLES[number]; // 'admin' | 'editor' | 'viewer'

const CONFIG = {
  apiUrl: 'https://api.example.com',
  timeout: 5000,
  retries: 3,
} as const;
// Type is { readonly apiUrl: "https://api.example.com"; readonly timeout: 5000; readonly retries: 3; }

// Instead of:
const CONFIG_BAD = {
  apiUrl: 'https://api.example.com',
  timeout: 5000,
  retries: 3,
};
// Type is { apiUrl: string; timeout: number; retries: number; }

// Tuple inference fix
function useTuple() {
  const pair = [1, 'hello'] as const; // [1, 'hello'] not (string | number)[]
  return pair;
}

// Enum alternative with const assertion
const STATUS_CODES = {
  OK: 200,
  NOT_FOUND: 404,
  ERROR: 500,
} as const;

type StatusCode = typeof STATUS_CODES[keyof typeof STATUS_CODES]; // 200 | 404 | 500

// Nested const assertion
const API_ENDPOINTS = {
  users: {
    list: '/users',
    create: '/users',
    get: (id: string) => `/users/${id}`,
  },
  orders: {
    list: '/orders',
    create: '/orders',
  },
} as const;
```

> **VIP EXPERT NOTE:** `as const` creates readonly literal types. Use it for:
> configuration objects, tuple types, and enum alternatives. It prevents accidental
> widening of string literals to `string` or number literals to `number`. Combined
> with `typeof`, it creates powerful derived types that stay in sync with your data.

---

## 🔷 SATISFIES OPERATOR (TypeScript 4.9+)

```typescript
// ✅ Ensure type matches while preserving literal types
type Routes = {
  home: '/';
  about: '/about';
  contact: '/contact';
};

const routes = {
  home: '/',
  about: '/about',
  contact: '/contact',
  // blog: '/blog', // Error: 'blog' not in Routes
} satisfies Routes;

// routes.home is '/' not string
const home: '/' = routes.home; // ✅ OK

// Satisfies with union types
type Color = 'red' | 'green' | 'blue';

const theme = {
  primary: 'blue',
  secondary: 'red',
} satisfies Record<string, Color>;

// Satisfies for configuration validation
type Config = {
  port: number;
  host: string;
  ssl: boolean;
};

const config = {
  port: 3000,
  host: 'localhost',
  ssl: false,
  // debug: true, // Error: 'debug' not in Config
} satisfies Config;
```

> **VIP EXPERT NOTE:** `satisfies` validates that a value conforms to a type without
> widening it. Unlike type annotations (`: Type`), it preserves literal types.
> Use it for: configuration objects, route definitions, and theme constants.
> It's the best of both worlds: validation + inference.

---

## 🔷 ZOD + TYPESCRIPT INTEGRATION (Runtime + Compile-Time Safety)

> **VIP EXPERT NOTE:** TypeScript types disappear at runtime. Zod gives you
> runtime validation that *generates* TypeScript types — single source of truth.

```typescript
import { z } from 'zod';

// Define schema ONCE — get both runtime validation AND TypeScript type
const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(100),
  email: z.string().email(),
  role: z.enum(['admin', 'editor', 'viewer']),
  metadata: z.record(z.unknown()).optional(),
  createdAt: z.string().datetime(),
});

// Derive TypeScript type from schema (never goes out of sync!)
type User = z.infer<typeof UserSchema>;

// API handler with automatic validation:
async function handleCreateUser(req: Request): Promise<Response> {
  const result = UserSchema.safeParse(req.body);
  
  if (!result.success) {
    // Zod provides detailed, field-level error messages
    return Response.json(
      { errors: result.error.flatten().fieldErrors },
      { status: 422 }
    );
  }
  
  // result.data is fully typed as User — no casting needed
  const user = await db.users.create(result.data);
  return Response.json(user, { status: 201 });
}

// Transform + validate in one step:
const CreateUserSchema = UserSchema.omit({ id: true, createdAt: true }).extend({
  password: z.string().min(8).regex(/[A-Z]/, 'Must contain uppercase'),
});
type CreateUserInput = z.infer<typeof CreateUserSchema>;

// Coerce types
const QuerySchema = z.object({
  page: z.coerce.number().default(1),
  limit: z.coerce.number().max(100).default(20),
  search: z.string().optional(),
});

// Discriminated unions in Zod
const PaymentSchema = z.discriminatedUnion('type', [
  z.object({ type: z.literal('credit_card'), last4: z.string() }),
  z.object({ type: z.literal('paypal'), email: z.string().email() }),
  z.object({ type: z.literal('crypto'), address: z.string() }),
]);
```

> **VIP EXPERT NOTE:** Zod is the gold standard for schema validation in TypeScript.
> It provides: type inference, composable schemas, detailed error messages, and
> transformations. Use `safeParse` for error handling, `parse` for throwing on
> invalid data. The discriminated union feature provides the same type narrowing
> as raw TypeScript.

---

## 🔷 VIP EXPERT NOTE: ERROR HANDLING WITH TYPE NARROWING

```typescript
// The typed error pattern — eliminates "error is unknown" frustration:

// Step 1: Define error hierarchy
class AppError extends Error {
  constructor(message: string, public readonly code: string) {
    super(message);
    this.name = this.constructor.name;
  }
}

class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} ${id} not found`, 'NOT_FOUND');
  }
}

class ValidationError extends AppError {
  constructor(public readonly fields: Record<string, string>) {
    super('Validation failed', 'VALIDATION_ERROR');
  }
}

class UnauthorizedError extends AppError {
  constructor(message: string = 'Unauthorized') {
    super(message, 'UNAUTHORIZED');
  }
}

// Step 2: Use Result type instead of throwing
type Result<T, E extends Error = AppError> =
  | { success: true; data: T }
  | { success: false; error: E };

function ok<T>(data: T): Result<T, never> {
  return { success: true, data };
}

function fail<E extends Error>(error: E): Result<never, E> {
  return { success: false, error };
}

// Step 3: Use in functions
async function getUser(id: string): Promise<Result<User, NotFoundError>> {
  const user = await db.users.findById(id);
  if (!user) return fail(new NotFoundError('User', id));
  return ok(user);
}

async function createUser(data: CreateUserInput): Promise<Result<User, ValidationError>> {
  const validation = validateUser(data);
  if (!validation.valid) {
    return fail(new ValidationError(validation.errors));
  }
  const user = await db.users.create(data);
  return ok(user);
}

// Step 4: Caller gets exhaustive type narrowing
async function handleRequest() {
  const result = await getUser('123');
  if (!result.success) {
    // TypeScript knows: result.error is NotFoundError
    console.log(result.error.code); // 'NOT_FOUND'
    return;
  }
  // TypeScript knows: result.data is User
  console.log(result.data.name);
}

// Helper for unwrap with fallback
function unwrapOr<T>(result: Result<T, Error>, fallback: T): T {
  return result.success ? result.data : fallback;
}

// Helper for mapping results
function mapResult<T, U, E extends Error>(
  result: Result<T, E>,
  fn: (data: T) => U
): Result<U, E> {
  return result.success ? ok(fn(result.data)) : result;
}
```

> **VIP EXPERT NOTE:** The Result pattern eliminates try/catch hell and makes
> errors explicit in the type signature. TypeScript's narrowing works perfectly
> with the discriminated union. This is how Rust and other modern languages
> handle errors — explicit, type-safe, and composable.

---

## 🔷 VIP EXPERT NOTE: MODERN TYPESCRIPT FEATURES (4.9+)

```typescript
// 1. satisfies operator (4.9) — type-check without widening
const routes = {
  home: '/',
  about: '/about',
} satisfies Record<string, string>;
// routes.home is '/' (literal), not string

// 2. const type parameters (5.0) — infer literal types
function createConfig<const T extends Record<string, unknown>>(config: T): T {
  return config;
}
const config = createConfig({ port: 3000, host: 'localhost' });
// config.port is 3000 (literal), not number

// 3. using keyword with Disposable (5.2) — automatic cleanup
class TempFile implements Disposable {
  constructor(public readonly path: string) {
    writeFileSync(path, '');
  }
  [Symbol.dispose](): void {
    unlinkSync(this.path);
  }
}

function processData() {
  using tmpFile = new TempFile('/tmp/data.tmp');
  // Use tmpFile...
  // Automatically cleaned up when scope exits (even on throw!)
}

// 4. NoInfer utility type (5.4) — prevent unwanted inference
function createEvent<T>(type: NoInfer<T>, handler: (data: T) => void): void {
  // T is not inferred from 'type' parameter — only from 'handler'
}

// 5. Decorators (5.0 with metadata)
function logged(target: any, key: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  descriptor.value = function(...args: any[]) {
    console.log(`Calling ${key} with`, args);
    return original.apply(this, args);
  };
}

class Example {
  @logged
  greet(name: string) {
    return `Hello, ${name}`;
  }
}

// 6. Import attributes (for JSON imports)
import config from './config.json' with { type: 'json' };
```

> **VIP EXPERT NOTE:** TypeScript evolves rapidly. Stay current with new features:
> `satisfies` (4.9) is transformative for config objects. `const` type parameters
> (5.0) preserve literal types in generics. The `using` keyword (5.2) brings
> explicit resource management. Enable `target: ESNext` to use latest features.

---

*This module provides TypeScript-specific patterns and best practices for production applications.*
