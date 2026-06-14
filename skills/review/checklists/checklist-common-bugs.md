# Common Bugs Checklist

Language-specific bugs and issues to watch for during code review.

## Universal Issues

### Logic Errors
- [ ] Off-by-one errors in loops and array access
- [ ] Incorrect boolean logic (De Morgan's law violations)
- [ ] Missing null/undefined checks
- [ ] Race conditions in concurrent code
- [ ] Incorrect comparison operators (== vs ===, = vs ==)
- [ ] Integer overflow/underflow
- [ ] Floating point comparison issues

### Resource Management
- [ ] Memory leaks (unclosed connections, listeners)
- [ ] File handles not closed
- [ ] Database connections not released
- [ ] Event listeners not removed
- [ ] Timers/intervals not cleared

### Error Handling
- [ ] Swallowed exceptions (empty catch blocks)
- [ ] Generic exception handling hiding specific errors
- [ ] Missing error propagation
- [ ] Incorrect error types thrown
- [ ] Missing finally/cleanup blocks

## TypeScript/JavaScript

### Type Issues
```typescript
// ❌ Using any defeats type safety
function process(data: any) { return data.value; }

// ✅ Use proper types
interface Data { value: string; }
function process(data: Data) { return data.value; }
```

### Async/Await Pitfalls
```typescript
// ❌ Missing await
async function fetch() {
  const data = fetchData();  // Missing await!
  return data.json();
}

// ❌ Unhandled promise rejection
async function risky() {
  const result = await fetchData();  // No try-catch
  return result;
}

// ✅ Proper error handling
async function safe() {
  try {
    const result = await fetchData();
    return result;
  } catch (error) {
    console.error('Fetch failed:', error);
    throw error;
  }
}
```

### React Specific

#### Hooks Rule Violations
```tsx
// ❌ Conditional Hooks call — violates Hooks rules
function BadComponent({ show }) {
  if (show) {
    const [value, setValue] = useState(0);  // Error!
  }
  return <div>...</div>;
}

// ✅ Hooks must be called unconditionally at the top level
function GoodComponent({ show }) {
  const [value, setValue] = useState(0);
  if (!show) return null;
  return <div>{value}</div>;
}

// ❌ Calling Hooks in a loop
function BadLoop({ items }) {
  items.forEach(item => {
    const [selected, setSelected] = useState(false);  // Error!
  });
}

// ✅ Lift state up or use different data structure
function GoodLoop({ items }) {
  const [selectedIds, setSelectedIds] = useState<Set<string>>(new Set());
  return items.map(item => (
    <Item key={item.id} selected={selectedIds.has(item.id)} />
  ));
}
```

#### Common useEffect Mistakes
```tsx
// ❌ Incomplete dependency array — stale closure
function StaleClosureExample({ userId, onSuccess }) {
  const [data, setData] = useState(null);
  useEffect(() => {
    fetchData(userId).then(result => {
      setData(result);
      onSuccess(result);  // onSuccess may be stale!
    });
  }, [userId]);  // Missing onSuccess dependency
}

// ✅ Complete dependency array
useEffect(() => {
  fetchData(userId).then(result => {
    setData(result);
    onSuccess(result);
  });
}, [userId, onSuccess]);

// ❌ Infinite loop — updating dependency inside effect
function InfiniteLoop() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    setCount(count + 1);  // Triggers re-render, which triggers effect again
  }, [count]);  // Infinite loop!
}

// ❌ Missing cleanup — memory leak
function MemoryLeak({ userId }) {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetchUser(userId).then(setUser);  // Still calls setUser after unmount
  }, [userId]);
}

// ✅ Proper cleanup
function NoLeak({ userId }) {
  const [user, setUser] = useState(null);
  useEffect(() => {
    let cancelled = false;
    fetchUser(userId).then(data => {
      if (!cancelled) setUser(data);
    });
    return () => { cancelled = true; };
  }, [userId]);
}

// ❌ useEffect for derived state (anti-pattern)
function BadDerived({ items }) {
  const [total, setTotal] = useState(0);
  useEffect(() => {
    setTotal(items.reduce((a, b) => a + b.price, 0));
  }, [items]);  // Unnecessary effect + extra render
}

// ✅ Calculate directly or use useMemo
function GoodDerived({ items }) {
  const total = useMemo(
    () => items.reduce((a, b) => a + b.price, 0),
    [items]
  );
}

// ❌ useEffect for event response
function BadEvent() {
  const [query, setQuery] = useState('');
  useEffect(() => {
    if (query) logSearch(query);  // Should be in event handler
  }, [query]);
}

// ✅ Side effects in event handlers
function GoodEvent() {
  const handleSearch = (q: string) => {
    setQuery(q);
    logSearch(q);
  };
}
```

#### useMemo / useCallback Misuse
```tsx
// ❌ Over-optimization — constants don't need memo
function OverOptimized() {
  const config = useMemo(() => ({ api: '/v1' }), []);  // Pointless
  const noop = useCallback(() => {}, []);  // Pointless
}

// ❌ Empty dependency useMemo (may hide bugs)
function EmptyDeps({ user }) {
  const greeting = useMemo(() => `Hello ${user.name}`, []);
  // greeting doesn't update when user changes!
}

// ❌ useCallback dependencies always change
function UselessCallback({ data }) {
  const process = useCallback(() => {
    return data.map(transform);
  }, [data]);  // If data is a new reference every time, completely ineffective
}

// ❌ useMemo/useCallback without React.memo
function Parent() {
  const data = useMemo(() => compute(), []);
  const handler = useCallback(() => {}, []);
  return <Child data={data} onClick={handler} />;
  // Child isn't using React.memo, these optimizations are meaningless
}

// ✅ Correct optimization combination
const MemoChild = React.memo(function Child({ data, onClick }) {
  return <button onClick={onClick}>{data}</button>;
});

function Parent() {
  const data = useMemo(() => expensiveCompute(), [dep]);
  const handler = useCallback(() => {}, []);
  return <MemoChild data={data} onClick={handler} />;
}
```

#### Component Design Issues
```tsx
// ❌ Defining component inside another component
function Parent() {
  // Creates new Child function every render, causing full remount
  const Child = () => <div>child</div>;
  return <Child />;
}

// ✅ Component definition outside
const Child = () => <div>child</div>;
function Parent() {
  return <Child />;
}

// ❌ Props always new references — breaks memo
function BadProps() {
  return (
    <MemoComponent
      style={{ color: 'red' }}      // New object every render
      onClick={() => handle()}       // New function every render
      items={data.filter(x => x)}    // New array every render
    />
  );
}

// ❌ Directly modifying props
function MutateProps({ user }) {
  user.name = 'Changed';  // Never do this!
  return <div>{user.name}</div>;
}
```

#### Server Component Errors (React 19+)
```tsx
// ❌ Using client APIs in Server Component
// app/page.tsx (default is Server Component)
export default function Page() {
  const [count, setCount] = useState(0);  // Error!
  useEffect(() => {}, []);  // Error!
  return <button onClick={() => {}}>Click</button>;  // Error!
}

// ✅ Move interaction logic to Client Component
// app/counter.tsx
'use client';
export function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// app/page.tsx
import { Counter } from './counter';
export default async function Page() {
  const data = await fetchData();  // Server Components can directly await
  return <Counter initialCount={data.count} />;
}

// ❌ Marking parent component 'use client', making entire subtree client-side
// layout.tsx
'use client';  // Bad idea! All child components become client components
export default function Layout({ children }) { ... }
```

#### Testing Common Mistakes
```tsx
// ❌ Using container queries
const { container } = render(<Component />);
const button = container.querySelector('button');  // Not recommended

// ✅ Use screen and semantic queries
render(<Component />);
const button = screen.getByRole('button', { name: /submit/i });

// ❌ Using fireEvent
fireEvent.click(button);

// ✅ Use userEvent
await userEvent.click(button);

// ❌ Testing implementation details
expect(component.state.isOpen).toBe(true);

// ✅ Test behavior
expect(screen.getByRole('dialog')).toBeVisible();

// ❌ Waiting for synchronous query
await screen.getByText('Hello');  // getBy is synchronous

// ✅ Use findBy for async
await screen.findByText('Hello');  // findBy waits
```

### React Common Mistakes Checklist
- [ ] Hooks not called at top level (in conditions/loops)
- [ ] useEffect dependency array incomplete
- [ ] useEffect missing cleanup function
- [ ] useEffect used for derived state calculation
- [ ] useMemo/useCallback overused
- [ ] useMemo/useCallback not paired with React.memo
- [ ] Defining child components inside components
- [ ] Props are new object/function references (when passed to memo components)
- [ ] Directly modifying props
- [ ] Lists missing key or using index as key
- [ ] Server Components using client APIs
- [ ] 'use client' on parent component making entire tree client-side
- [ ] Testing uses container queries instead of screen
- [ ] Testing implementation details instead of behavior

### React 19 Actions & Forms Errors

```tsx
// === useActionState Mistakes ===

// ❌ Calling setState directly in Action instead of returning state
const [state, action] = useActionState(async (prev, formData) => {
  setSomeState(newValue);  // Wrong! Should return new state
}, initialState);

// ✅ Return new state
const [state, action] = useActionState(async (prev, formData) => {
  const result = await submitForm(formData);
  return { ...prev, data: result };  // Return new state
}, initialState);

// ❌ Forgetting to handle isPending
const [state, action] = useActionState(submitAction, null);
return <button>Submit</button>;  // User can click repeatedly

// ✅ Use isPending to disable button
const [state, action, isPending] = useActionState(submitAction, null);
return <button disabled={isPending}>Submit</button>;

// === useFormStatus Errors ===

// ❌ Calling useFormStatus at form level
function Form() {
  const { pending } = useFormStatus();  // Always undefined!
  return <form><button disabled={pending}>Submit</button></form>;
}

// ✅ Call in child component
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>Submit</button>;
}
function Form() {
  return <form><SubmitButton /></form>;
}

// === useOptimistic Errors ===

// ❌ Using for critical business operations
function PaymentButton() {
  const [optimisticPaid, setPaid] = useOptimistic(false);
  const handlePay = async () => {
    setPaid(true);  // Dangerous: Shows paid but may fail
    await processPayment();
  };
}

// ❌ No handling of rollback UI state
const [optimisticLikes, addLike] = useOptimistic(likes);
// After failure UI rolls back, user may be confused why likes disappeared

// ✅ Provide failure feedback
const handleLike = async () => {
  addLike(1);
  try {
    await likePost();
  } catch {
    toast.error('Like failed, please try again');  // Notify user
  }
};
```

### React 19 Forms Checklist
- [ ] useActionState returns new state instead of calling setState
- [ ] useActionState correctly uses isPending to disable submit
- [ ] useFormStatus called in form child component
- [ ] useOptimistic not used for critical operations (payments, deletions, etc.)
- [ ] useOptimistic has user feedback on failure
- [ ] Server Actions correctly marked 'use server'

### Suspense & Streaming Errors

```tsx
// === Suspense Boundary Mistakes ===

// ❌ One Suspense for entire page — slow content blocks fast content
function BadPage() {
  return (
    <Suspense fallback={<FullPageLoader />}>
      <FastHeader />      {/* Fast */}
      <SlowMainContent /> {/* Slow — blocks entire page */}
      <FastFooter />      {/* Fast */}
    </Suspense>
  );
}

// ✅ Independent boundaries, don't block each other
function GoodPage() {
  return (
    <>
      <FastHeader />
      <Suspense fallback={<ContentSkeleton />}>
        <SlowMainContent />
      </Suspense>
      <FastFooter />
    </>
  );
}

// ❌ No Error Boundary
function NoErrorHandling() {
  return (
    <Suspense fallback={<Loading />}>
      <DataFetcher />  {/* Error causes blank screen */}
    </Suspense>
  );
}

// ✅ Error Boundary + Suspense
function WithErrorHandling() {
  return (
    <ErrorBoundary fallback={<ErrorFallback />}>
      <Suspense fallback={<Loading />}>
        <DataFetcher />
      </Suspense>
    </ErrorBoundary>
  );
}

// === use() Hook Mistakes ===

// ❌ Creating Promise outside component (new Promise every render)
function BadUse() {
  const data = use(fetchData());  // Creates new Promise every render!
  return <div>{data}</div>;
}

// ✅ Create in parent, pass via props
function Parent() {
  const dataPromise = useMemo(() => fetchData(), []);
  return <Child dataPromise={dataPromise} />;
}
function Child({ dataPromise }) {
  const data = use(dataPromise);
  return <div>{data}</div>;
}

// === Next.js Streaming Errors ===

// ❌ Awaiting slow data in layout.tsx — blocks all child pages
// app/layout.tsx
export default async function Layout({ children }) {
  const config = await fetchSlowConfig();  // Blocks entire app!
  return <ConfigProvider value={config}>{children}</ConfigProvider>;
}

// ✅ Put slow data at page level or use Suspense
// app/layout.tsx
export default function Layout({ children }) {
  return (
    <Suspense fallback={<ConfigSkeleton />}>
      <ConfigProvider>{children}</ConfigProvider>
    </Suspense>
  );
}
```

### Suspense Checklist
- [ ] Slow content has independent Suspense boundaries
- [ ] Each Suspense has corresponding Error Boundary
- [ ] fallback is a meaningful skeleton (not just a spinner)
- [ ] use() Promises not created during render
- [ ] Not awaiting slow data in layout
- [ ] Nesting level doesn't exceed 3 levels

### TanStack Query Errors

```tsx
// === Query Configuration Mistakes ===

// ❌ queryKey doesn't include query parameters
function BadQuery({ userId, filters }) {
  const { data } = useQuery({
    queryKey: ['users'],  // Missing userId and filters!
    queryFn: () => fetchUsers(userId, filters),
  });
  // Data won't update when userId or filters change
}

// ✅ queryKey includes all parameters affecting data
function GoodQuery({ userId, filters }) {
  const { data } = useQuery({
    queryKey: ['users', userId, filters],
    queryFn: () => fetchUsers(userId, filters),
  });
}

// ❌ staleTime: 0 causes excessive requests
const { data } = useQuery({
  queryKey: ['data'],
  queryFn: fetchData,
  // Default staleTime: 0, will refetch on every mount/window focus
});

// ✅ Set reasonable staleTime
const { data } = useQuery({
  queryKey: ['data'],
  queryFn: fetchData,
  staleTime: 5 * 60 * 1000,  // Won't auto-refetch within 5 minutes
});

// === useSuspenseQuery Errors ===

// ❌ useSuspenseQuery + enabled (not supported)
const { data } = useSuspenseQuery({
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId),
  enabled: !!userId,  // Wrong! useSuspenseQuery doesn't support enabled
});

// ✅ Conditional rendering implementation
function UserQuery({ userId }) {
  const { data } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });
  return <UserProfile user={data} />;
}

function Parent({ userId }) {
  if (!userId) return <SelectUser />;
  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserQuery userId={userId} />
    </Suspense>
  );
}

// === Mutation Errors ===

// ❌ Not invalidating queries after Mutation success
const mutation = useMutation({
  mutationFn: updateUser,
  // Forgot to invalidate, UI shows stale data
});

// ✅ Invalidate related queries on success
const mutation = useMutation({
  mutationFn: updateUser,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['users'] });
  },
});

// ❌ Optimistic update doesn't handle rollback
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    queryClient.setQueryData(['todos'], (old) => [...old, newTodo]);
    // Didn't save old data, can't rollback on failure!
  },
});

// ✅ Complete optimistic update
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    await queryClient.cancelQueries({ queryKey: ['todos'] });
    const previous = queryClient.getQueryData(['todos']);
    queryClient.setQueryData(['todos'], (old) => [...old, newTodo]);
    return { previous };
  },
  onError: (err, newTodo, context) => {
    queryClient.setQueryData(['todos'], context.previous);
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] });
  },
});

// === v5 Migration Errors ===

// ❌ Using deprecated API
const { data, isLoading } = useQuery(['key'], fetchFn);  // v4 syntax

// ✅ v5 single object parameter
const { data, isPending } = useQuery({
  queryKey: ['key'],
  queryFn: fetchFn,
});

// ❌ Confusing isPending and isLoading
if (isLoading) return <Spinner />;
// In v5, isLoading = isPending && isFetching

// ✅ Choose based on intent
if (isPending) return <Spinner />;  // No cached data
// Or
if (isFetching) return <Refreshing />;  // Refreshing in background
```

### TanStack Query Checklist
- [ ] queryKey includes all parameters affecting data
- [ ] Reasonable staleTime set (not default 0)
- [ ] useSuspenseQuery doesn't use enabled
- [ ] Invalidating related queries after Mutation success
- [ ] Optimistic updates have complete rollback logic
- [ ] v5 uses single object parameter syntax
- [ ] Understanding isPending vs isLoading vs isFetching

### TypeScript/JavaScript Common Mistakes
- [ ] `==` instead of `===`
- [ ] Modifying array/object during iteration
- [ ] `this` context lost in callbacks
- [ ] Missing `key` prop in lists
- [ ] Closure capturing loop variable
- [ ] parseInt without radix parameter

## Vue 3

### Reactivity Loss
```vue
<!-- ❌ Destructuring reactive loses reactivity -->
<script setup>
const state = reactive({ count: 0 })
const { count } = state  // count is not reactive!
</script>

<!-- ✅ Use toRefs -->
<script setup>
const state = reactive({ count: 0 })
const { count } = toRefs(state)  // count.value is reactive
</script>
```

### Props Reactivity Passing
```vue
<!-- ❌ Passing props values to composable loses reactivity -->
<script setup>
const props = defineProps<{ id: string }>()
const { data } = useFetch(props.id)  // Won't refetch when id changes!
</script>

<!-- ✅ Use toRef or getter -->
<script setup>
const props = defineProps<{ id: string }>()
const { data } = useFetch(() => props.id)  // Getter maintains reactivity
// Or
const { data } = useFetch(toRef(props, 'id'))
</script>
```

### Watch Cleanup
```vue
<!-- ❌ Async watch without cleanup, causes race conditions -->
<script setup>
watch(id, async (newId) => {
  const data = await fetchData(newId)
  result.value = data  // Old request may overwrite new result!
})
</script>

<!-- ✅ Use onCleanup to cancel old requests -->
<script setup>
watch(id, async (newId, _, onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort())

  const data = await fetchData(newId, controller.signal)
  result.value = data
})
</script>
```

### Computed Side Effects
```vue
<!-- ❌ Modifying other state in computed -->
<script setup>
const total = computed(() => {
  sideEffect.value++  // Side effect! Executes every access
  return items.value.reduce((a, b) => a + b, 0)
})
</script>

<!-- ✅ computed only for pure calculations -->
<script setup>
const total = computed(() => {
  return items.value.reduce((a, b) => a + b, 0)
})
// Side effects in watch
watch(total, () => { sideEffect.value++ })
</script>
```

### Template Common Mistakes
```vue
<!-- ❌ v-if and v-for on same element (v-if has higher priority) -->
<template>
  <div v-for="item in items" v-if="item.visible" :key="item.id">
    {{ item.name }}
  </div>
</template>

<!-- ✅ Use computed or template wrapper -->
<template>
  <template v-for="item in items" :key="item.id">
    <div v-if="item.visible">{{ item.name }}</div>
  </template>
</template>
```

### Common Mistakes
- [ ] Destructuring reactive object loses reactivity
- [ ] Props not maintaining reactivity when passed to composable
- [ ] Watch async callbacks missing cleanup function
- [ ] Side effects in computed
- [ ] v-for using index as key (when list will reorder)
- [ ] v-if and v-for on same element
- [ ] defineProps not using TypeScript type declarations
- [ ] withDefaults object defaults not using factory functions
- [ ] Directly modifying props (instead of emit)
- [ ] watchEffect dependencies unclear causing over-triggering

## Python

### Mutable Default Arguments
```python
# ❌ Bug: List shared across all calls
def add_item(item, items=[]):
    items.append(item)
    return items

# ✅ Correct
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

### Exception Handling
```python
# ❌ Catching everything, including KeyboardInterrupt
try:
    risky_operation()
except:
    pass

# ✅ Catch specific exceptions
try:
    risky_operation()
except ValueError as e:
    logger.error(f"Invalid value: {e}")
    raise
```

### Class Attributes
```python
# ❌ Shared mutable class attribute
class User:
    permissions = []  # Shared across all instances!

# ✅ Initialize in __init__
class User:
    def __init__(self):
        self.permissions = []
```

### Common Mistakes
- [ ] Using `is` instead of `==` for value comparison
- [ ] Forgetting `self` parameter in methods
- [ ] Modifying list while iterating
- [ ] String concatenation in loops (use join)
- [ ] Not closing files (use `with` statement)

## Rust

### Ownership and Borrowing

```rust
// ❌ Use after move
let s = String::from("hello");
let s2 = s;
println!("{}", s);  // Error: s was moved

// ✅ Clone if needed (but consider if clone is necessary)
let s = String::from("hello");
let s2 = s.clone();
println!("{}", s);  // OK

// ❌ Using clone() to bypass borrow checker (anti-pattern)
fn process(data: &Data) {
    let owned = data.clone();  // Unnecessary clone
    do_something(owned);
}

// ✅ Correct borrowing usage
fn process(data: &Data) {
    do_something(data);  // Pass reference
}

// ❌ Storing references in struct (usually a bad idea)
struct Parser<'a> {
    input: &'a str,  // Complicates lifetimes
    position: usize,
}

// ✅ Use owned data
struct Parser {
    input: String,  // Owns data, simplifies lifetimes
    position: usize,
}

// ❌ Modifying collection while iterating
let mut vec = vec![1, 2, 3];
for item in &vec {
    vec.push(*item);  // Error: cannot borrow as mutable
}

// ✅ Collect into new collection
let vec = vec![1, 2, 3];
let new_vec: Vec<_> = vec.iter().map(|x| x * 2).collect();
```

### Unsafe Code Review

```rust
// ❌ unsafe without safety comment
unsafe {
    ptr::write(dest, value);
}

// ✅ Must have SAFETY comment explaining invariants
// SAFETY: dest pointer obtained from Vec::as_mut_ptr(), guaranteed:
// 1. Pointer is valid and properly aligned
// 2. Target memory not borrowed by other references
// 3. Write won't exceed allocated capacity
unsafe {
    ptr::write(dest, value);
}

// ❌ unsafe fn without # Safety documentation
pub unsafe fn from_raw_parts(ptr: *mut T, len: usize) -> Self { ... }

// ✅ Must document safety contract
/// Creates a new instance from raw parts.
///
/// # Safety
///
/// - `ptr` must have been allocated via `GlobalAlloc`
/// - `len` must be less than or equal to the allocated capacity
/// - The caller must ensure no other references to the memory exist
pub unsafe fn from_raw_parts(ptr: *mut T, len: usize) -> Self { ... }

// ❌ Cross-module unsafe invariants
mod a {
    pub fn set_flag() { FLAG = true; }  // Safe code affects unsafe
}
mod b {
    pub unsafe fn do_thing() {
        if FLAG { /* assumes FLAG means something */ }
    }
}

// ✅ Encapsulate unsafe boundaries in single module
mod safe_wrapper {
    // All unsafe logic in one module
    // Expose safe API outside
}
```

### Async/Concurrency

```rust
// ❌ Blocking in async context
async fn bad_fetch(url: &str) -> Result<String> {
    let resp = reqwest::blocking::get(url)?;  // Blocks entire runtime!
    Ok(resp.text()?)
}

// ✅ Use async version
async fn good_fetch(url: &str) -> Result<String> {
    let resp = reqwest::get(url).await?;
    Ok(resp.text().await?)
}

// ❌ Holding Mutex across .await
async fn bad_lock(mutex: &Mutex<Data>) {
    let guard = mutex.lock().unwrap();
    some_async_op().await;  // Holding lock across await!
    drop(guard);
}

// ✅ Minimize lock hold time
async fn good_lock(mutex: &Mutex<Data>) {
    let data = {
        let guard = mutex.lock().unwrap();
        guard.clone()  // Get data and release lock immediately
    };
    some_async_op().await;
    // Process data
}

// ❌ Using std::sync::Mutex in async function
async fn bad_async_mutex(mutex: &std::sync::Mutex<Data>) {
    let _guard = mutex.lock().unwrap();  // May deadlock
    tokio::time::sleep(Duration::from_secs(1)).await;
}

// ✅ Use tokio::sync::Mutex (if must cross await)
async fn good_async_mutex(mutex: &tokio::sync::Mutex<Data>) {
    let _guard = mutex.lock().await;
    tokio::time::sleep(Duration::from_secs(1)).await;
}

// ❌ Forgetting Future is lazy
fn bad_spawn() {
    let future = async_operation();  // Not executed!
    // future dropped, nothing happens
}

// ✅ Must await or spawn
async fn good_spawn() {
    async_operation().await;  // Execute
    // Or
    tokio::spawn(async_operation());  // Execute in background
}

// ❌ spawn task missing 'static
async fn bad_spawn_lifetime(data: &str) {
    tokio::spawn(async {
        println!("{}", data);  // Error: data is not 'static
    });
}

// ✅ Use move or Arc
async fn good_spawn_lifetime(data: String) {
    tokio::spawn(async move {
        println!("{}", data);  // OK: Owns data
    });
}
```

### Error Handling

```rust
// ❌ Using unwrap/expect in production code
fn bad_parse(input: &str) -> i32 {
    input.parse().unwrap()  // panic!
}

// ✅ Properly propagate errors
fn good_parse(input: &str) -> Result<i32, ParseIntError> {
    input.parse()
}

// ❌ Swallowing error info
fn bad_error_handling() -> Result<()> {
    match operation() {
        Ok(v) => Ok(v),
        Err(_) => Err(anyhow!("operation failed"))  // Lost original error
    }
}

// ✅ Use context to add context
fn good_error_handling() -> Result<()> {
    operation().context("failed to perform operation")?;
    Ok(())
}

// ❌ Library code using anyhow (should use thiserror)
// lib.rs
pub fn parse_config(path: &str) -> anyhow::Result<Config> {
    // Caller can't distinguish error types
}

// ✅ Library code uses thiserror to define error types
#[derive(Debug, thiserror::Error)]
pub enum ConfigError {
    #[error("failed to read config file: {0}")]
    Io(#[from] std::io::Error),
    #[error("invalid config format: {0}")]
    Parse(#[from] serde_json::Error),
}

pub fn parse_config(path: &str) -> Result<Config, ConfigError> {
    // Caller can match different errors
}

// ❌ Ignoring must_use return value
fn bad_ignore_result() {
    some_fallible_operation();  // Warning: unused Result
}

// ✅ Explicitly handle or mark as ignored
fn good_handle_result() {
    let _ = some_fallible_operation();  // Explicitly ignore
    // Or
    some_fallible_operation().ok();  // Convert to Option
}
```

### Performance Traps

```rust
// ❌ Unnecessary collect
fn bad_process(items: &[i32]) -> i32 {
    items.iter()
        .filter(|x| **x > 0)
        .collect::<Vec<_>>()  // Unnecessary allocation
        .iter()
        .sum()
}

// ✅ Lazy iteration
fn good_process(items: &[i32]) -> i32 {
    items.iter()
        .filter(|x| **x > 0)
        .sum()
}

// ❌ Repeated allocation in loop
fn bad_loop() -> String {
    let mut result = String::new();
    for i in 0..1000 {
        result = result + &i.to_string();  // Re-allocates every iteration!
    }
    result
}

// ✅ Pre-allocate or use push_str
fn good_loop() -> String {
    let mut result = String::with_capacity(4000);  // Pre-allocate
    for i in 0..1000 {
        write!(result, "{}", i).unwrap();  // Append in place
    }
    result
}

// ❌ Overusing clone
fn bad_clone(data: &HashMap<String, Vec<u8>>) -> Vec<u8> {
    data.get("key").cloned().unwrap_or_default()
}

// ✅ Return reference or use Cow
fn good_ref(data: &HashMap<String, Vec<u8>>) -> &[u8] {
    data.get("key").map(|v| v.as_slice()).unwrap_or(&[])
}

// ❌ Passing large struct by value
fn bad_pass(data: LargeStruct) { ... }  // Copies entire struct

// ✅ Pass by reference
fn good_pass(data: &LargeStruct) { ... }

// ❌ Box<dyn Trait> for small known types
fn bad_trait_object() -> Box<dyn Iterator<Item = i32>> {
    Box::new(vec![1, 2, 3].into_iter())
}

// ✅ Use impl Trait
fn good_impl_trait() -> impl Iterator<Item = i32> {
    vec![1, 2, 3].into_iter()
}

// ❌ retain slower than filter+collect (certain scenarios)
vec.retain(|x| x.is_valid());  // O(n) but large constant factor

// ✅ If no in-place modification needed, consider filter
let vec: Vec<_> = vec.into_iter().filter(|x| x.is_valid()).collect();
```

### Lifetimes and References

```rust
// ❌ Returning reference to local variable
fn bad_return_ref() -> &str {
    let s = String::from("hello");
    &s  // Error: s will be dropped
}

// ✅ Return owned data or static reference
fn good_return_owned() -> String {
    String::from("hello")
}

// ❌ Over-generalized lifetimes
fn bad_lifetime<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
    x  // 'b not used
}

// ✅ Simplify lifetimes
fn good_lifetime(x: &str, _y: &str) -> &str {
    x  // Compiler infers automatically
}

// ❌ Struct holds multiple related references but lifetimes independent
struct Bad<'a, 'b> {
    name: &'a str,
    data: &'b [u8],  // Should usually be the same lifetime
}

// ✅ Related data uses same lifetime
struct Good<'a> {
    name: &'a str,
    data: &'a [u8],
}
```

### Rust Review Checklist

**Ownership and Borrowing**
- [ ] clone() is intentional, not bypassing borrow checker
- [ ] Avoid storing references in structs (unless necessary)
- [ ] Rc/Arc usage reasonable, no hidden unnecessary shared state
- [ ] No unnecessary RefCell (runtime checks vs compile-time)

**Unsafe Code**
- [ ] Every unsafe block has SAFETY comment
- [ ] unsafe fn has # Safety documentation
- [ ] Safety invariants clearly documented
- [ ] Unsafe boundaries kept as small as possible

**Async/Concurrency**
- [ ] No blocking in async context
- [ ] No holding std::sync locks across .await
- [ ] spawn'd tasks satisfy 'static constraint
- [ ] Futures properly awaited or spawned
- [ ] Lock ordering consistent (avoid deadlocks)

**Error Handling**
- [ ] Library uses thiserror, application uses anyhow
- [ ] Errors have enough context information
- [ ] No unwrap/expect in production code
- [ ] must_use return values properly handled

**Performance**
- [ ] Avoid unnecessary collect()
- [ ] Large data structures passed by reference
- [ ] String concatenation uses String::with_capacity or write!
- [ ] impl Trait preferred over Box<dyn Trait> (when possible)

## SQL

### Injection Vulnerabilities
```sql
-- ❌ String concatenation (SQL injection risk)
query = "SELECT * FROM users WHERE id = " + user_id

-- ✅ Parameterized queries
query = "SELECT * FROM users WHERE id = ?"
cursor.execute(query, (user_id,))
```

### Performance Issues
- [ ] Missing indexes on filtered/joined columns
- [ ] SELECT * instead of specific columns
- [ ] N+1 query patterns
- [ ] Missing LIMIT on large tables
- [ ] Inefficient subqueries vs JOINs

### Common Mistakes
- [ ] Not handling NULL comparisons correctly
- [ ] Missing transactions for related operations
- [ ] Incorrect JOIN types
- [ ] Case sensitivity issues
- [ ] Date/timezone handling errors

## API Design

### REST Issues
- [ ] Inconsistent resource naming
- [ ] Wrong HTTP methods (POST for idempotent operations)
- [ ] Missing pagination for list endpoints
- [ ] Incorrect status codes
- [ ] Missing rate limiting

### Data Validation
- [ ] Missing input validation
- [ ] Incorrect data type validation
- [ ] Missing length/range checks
- [ ] Not sanitizing user input
- [ ] Trusting client-side validation

## Testing

### Test Quality Issues
- [ ] Testing implementation details instead of behavior
- [ ] Missing edge case tests
- [ ] Flaky tests (non-deterministic)
- [ ] Tests with external dependencies
- [ ] Missing negative tests (error cases)
- [ ] Overly complex test setup
