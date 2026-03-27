# React Patterns — Professional Development Guide

> **VIP EXPERT NOTE:** React's component model enables powerful abstractions, but its
> flexibility can lead to subtle bugs — especially around hooks, effects, and state
> management. The patterns below represent battle-tested solutions from production
> applications handling millions of users.

---

## 🔷 CUSTOM HOOK PATTERN (Data Fetching with useAsync)

```tsx
// ✅ GOLD STANDARD: Custom hook with all states handled
// VIP NOTE: This hook properly handles cancellation, error typing, and all 4 states.
function useAsync<T>(asyncFn: () => Promise<T>, deps: React.DependencyList) {
  const [state, setState] = useState<
    | { status: 'idle' }
    | { status: 'loading' }
    | { status: 'success'; data: T }
    | { status: 'error'; error: Error }
  >({ status: 'idle' });

  useEffect(() => {
    let cancelled = false;
    setState({ status: 'loading' });

    asyncFn()
      .then((data) => { if (!cancelled) setState({ status: 'success', data }); })
      .catch((error) => {
        if (!cancelled) setState({
          status: 'error',
          error: error instanceof Error ? error : new Error(String(error)),
        });
      });

    return () => { cancelled = true; };
    // Note: deps is intentionally spread here because callers control the dependency
    // array. In production, consider using a linter plugin that understands this pattern,
    // or switch to React Query / SWR which handles this out of the box.
    // eslint-disable-next-line react-hooks/exhaustive-deps -- caller controls deps
  }, deps);

  return state;
}

// Usage — TypeScript knows exact shape in each branch
function UserPage({ userId }: { userId: string }) {
  const state = useAsync(() => fetchUser(userId), [userId]);

  if (state.status === 'idle' || state.status === 'loading') return <Spinner />;
  if (state.status === 'error') return <ErrorMessage error={state.error} />;
  return <UserProfile user={state.data} />; // TypeScript: data is User
}
```

> **VIP EXPERT NOTE:** Always use discriminated unions for async state. This pattern
> eliminates the impossible state problem (e.g., `isLoading && data !== null`) and gives
> TypeScript perfect narrowing. Consider React Query (TanStack Query) for production —
> it handles caching, deduplication, and background refetching automatically.

---

## 🔷 FORM STATE PATTERN (useReducer for Complex Forms)

```tsx
// ✅ CORRECT: useReducer for complex form state
type FormAction =
  | { type: 'SET_FIELD'; field: keyof FormData; value: string }
  | { type: 'SET_ERRORS'; errors: Partial<Record<keyof FormData, string>> }
  | { type: 'SUBMIT_START' }
  | { type: 'SUBMIT_SUCCESS' }
  | { type: 'SUBMIT_ERROR'; error: string };

interface FormState {
  data: { name: string; email: string; password: string };
  errors: Partial<Record<'name' | 'email' | 'password', string>>;
  isSubmitting: boolean;
  submitError: string | null;
}

function formReducer(state: FormState, action: FormAction): FormState {
  switch (action.type) {
    case 'SET_FIELD':
      return {
        ...state,
        data: { ...state.data, [action.field]: action.value },
        errors: { ...state.errors, [action.field]: undefined }, // clear error on change
      };
    case 'SET_ERRORS':
      return { ...state, errors: action.errors };
    case 'SUBMIT_START':
      return { ...state, isSubmitting: true, submitError: null };
    case 'SUBMIT_SUCCESS':
      return { ...state, isSubmitting: false };
    case 'SUBMIT_ERROR':
      return { ...state, isSubmitting: false, submitError: action.error };
    default:
      return state;
  }
}

function RegisterForm() {
  const [state, dispatch] = useReducer(formReducer, {
    data: { name: '', email: '', password: '' },
    errors: {},
    isSubmitting: false,
    submitError: null,
  });

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const errors = validateForm(state.data);
    if (Object.keys(errors).length > 0) {
      dispatch({ type: 'SET_ERRORS', errors });
      return;
    }
    dispatch({ type: 'SUBMIT_START' });
    try {
      await registerUser(state.data);
      dispatch({ type: 'SUBMIT_SUCCESS' });
    } catch (error) {
      dispatch({ type: 'SUBMIT_ERROR', error: String(error) });
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={state.data.name}
        onChange={e => dispatch({ type: 'SET_FIELD', field: 'name', value: e.target.value })}
        aria-invalid={!!state.errors.name}
        aria-describedby={state.errors.name ? 'name-error' : undefined}
      />
      {state.errors.name && <span id="name-error" role="alert">{state.errors.name}</span>}
      {/* ... other fields */}
      <button type="submit" disabled={state.isSubmitting}>
        {state.isSubmitting ? 'Registering...' : 'Register'}
      </button>
      {state.submitError && <ErrorBanner message={state.submitError} />}
    </form>
  );
}
```

> **VIP EXPERT NOTE:** useReducer shines when form state has interdependent fields
> or complex validation logic. For simple forms (3 fields or less), useState is fine.
> For complex enterprise forms, consider libraries like React Hook Form which provide
> built-in validation, error handling, and performance optimizations.

---

## 🔷 COMPOUND COMPONENT PATTERN (Tabs with Context)

```tsx
// ✅ For complex UI components — avoids deep prop drilling within component family
interface TabsContextValue {
  activeTab: string;
  setActiveTab: (id: string) => void;
}

const TabsContext = createContext<TabsContextValue | null>(null);

function useTabs() {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('Tab components must be used within <Tabs>');
  return ctx;
}

function Tabs({ defaultTab, children }: { defaultTab: string; children: React.ReactNode }) {
  const [activeTab, setActiveTab] = useState(defaultTab);
  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div role="tablist">{children}</div>
    </TabsContext.Provider>
  );
}

function Tab({ id, label }: { id: string; label: string }) {
  const { activeTab, setActiveTab } = useTabs();
  return (
    <button
      role="tab"
      aria-selected={activeTab === id}
      onClick={() => setActiveTab(id)}
    >
      {label}
    </button>
  );
}

function TabPanel({ id, children }: { id: string; children: React.ReactNode }) {
  const { activeTab } = useTabs();
  if (activeTab !== id) return null;
  return <div role="tabpanel">{children}</div>;
}

// Usage — clean, composable, no prop drilling
<Tabs defaultTab="overview">
  <Tab id="overview" label="Overview" />
  <Tab id="settings" label="Settings" />
  <TabPanel id="overview"><Overview /></TabPanel>
  <TabPanel id="settings"><Settings /></TabPanel>
</Tabs>
```

> **VIP EXPERT NOTE:** Compound components provide a flexible, declarative API for
> complex UI primitives. The key insight: parent component (Tabs) owns the state,
> children communicate via context. This pattern is used in Radix UI, Headless UI,
> and React Aria for accessible, composable components.

---

## 🔷 ERROR BOUNDARY (Class Component with Fallback)

```tsx
interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends React.Component<
  { children: React.ReactNode; fallback?: React.ComponentType<{ error: Error; reset: () => void }> },
  ErrorBoundaryState
> {
  state: ErrorBoundaryState = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    logger.error('React error boundary caught error', {
      error: error.message,
      stack: error.stack,
      componentStack: info.componentStack,
    });
  }

  reset = () => this.setState({ hasError: false, error: null });

  render() {
    if (this.state.hasError && this.state.error) {
      const FallbackComponent = this.props.fallback ?? DefaultErrorFallback;
      return <FallbackComponent error={this.state.error} reset={this.reset} />;
    }
    return this.props.children;
  }
}

function DefaultErrorFallback({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div role="alert">
      <h2>Something went wrong</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

> **VIP EXPERT NOTE:** Error boundaries ONLY catch errors during rendering, in lifecycle
> methods, and constructors. They do NOT catch errors in: event handlers, async code,
> server-side rendering, or errors thrown in the error boundary itself. For event
> handlers, use try/catch. For async errors, use useEffect error handling patterns.

---

## 🔷 DEBOUNCED VALUE HOOK

```tsx
function useDebouncedValue<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Usage: Debounce search input
function SearchInput() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<SearchResult[]>([]);
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
```

> **VIP EXPERT NOTE:** Debouncing prevents excessive API calls or expensive computations
> on every keystroke. The 300ms delay is a sweet spot — fast enough to feel responsive,
> slow enough to avoid thrashing. Combine with AbortController for race condition
> protection when the user types faster than the API responds.

---

## 🔷 PREVIOUS VALUE HOOK

```tsx
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>();

  useEffect(() => {
    ref.current = value;
  }, [value]);

  return ref.current;
}

// Usage: Detect when value changes
function Component({ userId }: { userId: string }) {
  const prevUserId = usePrevious(userId);

  useEffect(() => {
    if (prevUserId !== userId) {
      console.log(`User ID changed from ${prevUserId} to ${userId}`);
    }
  }, [userId, prevUserId]);
}
```

> **VIP EXPERT NOTE:** This hook captures the value from the previous render. It's
> useful for comparing props/state changes and triggering side effects conditionally.
> Note: On first render, it returns undefined. For complex change detection, consider
> a custom comparator: `usePrevious(value, deepEqual)`.

---

## 🔷 INTERSECTION OBSERVER HOOK

```tsx
function useIntersectionObserver(
  ref: RefObject<HTMLElement>,
  options?: IntersectionObserverInit
): boolean {
  const [isIntersecting, setIsIntersecting] = useState(false);

  // ⚠️ VIP NOTE: If caller passes inline options object, this will re-create
  // the observer on every render. Callers should useMemo their options:
  //   const options = useMemo(() => ({ threshold: 0.1 }), []);
  useEffect(() => {
    const element = ref.current;
    if (!element) return;

    const observer = new IntersectionObserver(([entry]) => {
      setIsIntersecting(entry.isIntersecting);
    }, options);

    observer.observe(element);
    return () => observer.disconnect();
  }, [ref, options]);

  return isIntersecting;
}

// Usage: Lazy load when visible
function LazyImage({ src, alt }: { src: string; alt: string }) {
  const ref = useRef<HTMLDivElement>(null);
  const isVisible = useIntersectionObserver(ref, { threshold: 0.1 });

  return (
    <div ref={ref} style={{ minHeight: 200 }}>
      {isVisible ? <img src={src} alt={alt} /> : <Placeholder />}
    </div>
  );
}
```

> **VIP EXPERT NOTE:** IntersectionObserver is more performant than scroll event
> listeners for detecting element visibility. It runs on the browser's compositor
> thread and doesn't block the main thread. Use for: lazy loading, infinite scroll,
> analytics (tracking impressions), and animating elements on scroll.

---

## 🔷 WINDOW SIZE HOOK

```tsx
function useWindowSize() {
  const [size, setSize] = useState({ 
    width: typeof window !== 'undefined' ? window.innerWidth : 0, 
    height: typeof window !== 'undefined' ? window.innerHeight : 0 
  });

  useEffect(() => {
    const handleResize = () => {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    };

    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return size;
}

// Usage: Responsive rendering
function Component() {
  const { width } = useWindowSize();
  const isMobile = width < 768;

  return isMobile ? <MobileLayout /> : <DesktopLayout />;
}
```

> **VIP EXPERT NOTE:** While useful, this hook runs on every resize event (potentially
> 60+ times/second during drag). For breakpoint detection, prefer useMediaQuery
> which uses matchMedia and only fires when crossing thresholds. Use useWindowSize
> only when you need the actual pixel dimensions.

---

## 🔷 MEDIA QUERY HOOK (SSR-Safe)

```tsx
// ✅ SSR-safe: doesn't call window on server, uses matchMedia for efficiency
function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(() => {
    // SSR-safe: check if window exists
    if (typeof window === 'undefined') return false;
    return window.matchMedia(query).matches;
  });

  useEffect(() => {
    const mediaQuery = window.matchMedia(query);
    const handler = (event: MediaQueryListEvent) => setMatches(event.matches);
    
    // Set initial value
    setMatches(mediaQuery.matches);
    
    mediaQuery.addEventListener('change', handler);
    return () => mediaQuery.removeEventListener('change', handler);
  }, [query]);

  return matches;
}

// Usage:
function Component() {
  const isMobile = useMediaQuery('(max-width: 768px)');
  const prefersReducedMotion = useMediaQuery('(prefers-reduced-motion: reduce)');
  const isDarkMode = useMediaQuery('(prefers-color-scheme: dark)');
  
  return isMobile ? <MobileLayout /> : <DesktopLayout />;
}
```

> **VIP EXPERT NOTE — useMediaQuery vs useWindowSize:**
> Prefer `useMediaQuery` over `useWindowSize` for responsive rendering.
> `matchMedia` is more performant (only fires on breakpoint changes),
> while `resize` fires continuously during drag (60+ times/second).

---

## 🔷 VIP EXPERT NOTE: REACT 19+ & SERVER COMPONENTS

```
Key patterns for modern React:

1. Server Components (default in Next.js App Router):
   - Components are server-rendered by default
   - Add 'use client' directive ONLY for components using hooks, event handlers, or browser APIs
   - Keep server components for data fetching, heavy computation

2. use() hook (React 19):
   - Replaces useEffect for data fetching in many cases
   - Works with Promises and Context
   - Integrates with Suspense boundaries

3. Form Actions (React 19):
   - Use action={} prop on <form> instead of onSubmit + e.preventDefault()
   - Server actions for form submissions that hit the server directly
   - useActionState() for form submission status

4. Key Decision Framework:
   ┌──────────────────────────────────────────────────┐
   │ Does it use useState, useEffect, onClick?        │
   │   YES → 'use client' component                   │
   │   NO  → Server component (default)               │
   │                                                  │
   │ Does it fetch data?                              │
   │   Server Component → async/await directly        │
   │   Client Component → React Query / SWR           │
   └──────────────────────────────────────────────────┘
```

> **VIP EXPERT NOTE:** The React ecosystem is shifting toward Server Components as
> the default. This reduces client-side JavaScript and improves initial page load.
> Learn to think in Server Components first, then add client directives only when
> necessary. The boundary between server and client is now a core architectural
> decision in React applications.

---

## 🔷 ADDITIONAL PATTERNS

### useOnClickOutside Hook

```tsx
function useOnClickOutside(
  ref: RefObject<HTMLElement>,
  handler: (event: MouseEvent | TouchEvent) => void
) {
  useEffect(() => {
    const listener = (event: MouseEvent | TouchEvent) => {
      if (!ref.current || ref.current.contains(event.target as Node)) {
        return;
      }
      handler(event);
    };

    document.addEventListener('mousedown', listener);
    document.addEventListener('touchstart', listener);
    return () => {
      document.removeEventListener('mousedown', listener);
      document.removeEventListener('touchstart', listener);
    };
  }, [ref, handler]);
}

// Usage: Close dropdown when clicking outside
function Dropdown() {
  const ref = useRef<HTMLDivElement>(null);
  const [isOpen, setIsOpen] = useState(false);
  
  useOnClickOutside(ref, () => setIsOpen(false));
  
  return (
    <div ref={ref}>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      {isOpen && <DropdownMenu />}
    </div>
  );
}
```

### useKeyPress Hook

```tsx
function useKeyPress(targetKey: string): boolean {
  const [keyPressed, setKeyPressed] = useState(false);

  useEffect(() => {
    const downHandler = ({ key }: KeyboardEvent) => {
      if (key === targetKey) setKeyPressed(true);
    };
    const upHandler = ({ key }: KeyboardEvent) => {
      if (key === targetKey) setKeyPressed(false);
    };

    window.addEventListener('keydown', downHandler);
    window.addEventListener('keyup', upHandler);
    return () => {
      window.removeEventListener('keydown', downHandler);
      window.removeEventListener('keyup', upHandler);
    };
  }, [targetKey]);

  return keyPressed;
}

// Usage: Close modal on Escape
function Modal({ onClose }: { onClose: () => void }) {
  const escapePressed = useKeyPress('Escape');
  
  useEffect(() => {
    if (escapePressed) onClose();
  }, [escapePressed, onClose]);
  
  return <div className="modal">...</div>;
}
```

---

*This module provides React-specific patterns and best practices for production applications.*
