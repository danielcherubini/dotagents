# Rust Code Review Guide

> Rust code review guide. The compiler catches memory safety issues, but reviewers need to focus on what the compiler can't detect—business logic, API design, performance, cancellation safety, and maintainability.

## Table of Contents

- [Ownership and Borrowing](#ownership-and-borrowing)
- [Unsafe Code Review](#unsafe-code-review-critical)
- [Async Code](#async-code)
- [Cancellation Safety](#cancellation-safety)
- [spawn vs await](#spawn-vs-await)
- [Error Handling](#error-handling)
- [Performance](#performance)
- [Trait Design](#trait-design)
- [Testing](#testing)
- [Macros](#macros)
- [Common Anti-Patterns](#common-anti-patterns)
- [Review Checklist](#rust-review-checklist)

---

## Ownership and Borrowing

### Avoid Unnecessary clone()

```rust
// ❌ clone() is "Rust's duct tape" — used to bypass the borrow checker
fn bad_process(data: &Data) -> Result<()> {
    let owned = data.clone();  // Why do we need clone?
    expensive_operation(owned)
}

// ✅ When reviewing, ask: Is clone necessary? Can we use borrowing?
fn good_process(data: &Data) -> Result<()> {
    expensive_operation(data)  // Pass reference
}

// ✅ If clone is truly needed, add a comment explaining why
fn justified_clone(data: &Data) -> Result<()> {
    // Clone needed: data will be moved to spawned task
    let owned = data.clone();
    tokio::spawn(async move {
        process(owned).await
    });
    Ok(())
}
```

### Arc<Mutex<T>> Usage

```rust
// ❌ Arc<Mutex<T>> may hide unnecessary shared state
struct BadService {
    cache: Arc<Mutex<HashMap<String, Data>>>,  // Do we really need sharing?
}

// ✅ Consider whether sharing is needed, or redesign to avoid it
struct GoodService {
    cache: HashMap<String, Data>,  // Single owner
}

// ✅ If concurrent access is truly needed, consider better data structures
use dashmap::DashMap;

struct ConcurrentService {
    cache: DashMap<String, Data>,  // Finer-grained locking
}
```

### Cow (Copy-on-Write) Pattern

```rust
use std::borrow::Cow;

// ❌ Always allocating new strings
fn bad_process_name(name: &str) -> String {
    if name.is_empty() {
        "Unknown".to_string()  // Allocation
    } else {
        name.to_string()  // Unnecessary allocation
    }
}

// ✅ Use Cow to avoid unnecessary allocations
fn good_process_name(name: &str) -> Cow<'_, str> {
    if name.is_empty() {
        Cow::Borrowed("Unknown")  // Static string, no allocation
    } else {
        Cow::Borrowed(name)  // Borrow original data
    }
}

// ✅ Only allocate when modification is needed
fn normalize_name(name: &str) -> Cow<'_, str> {
    if name.chars().any(|c| c.is_uppercase()) {
        Cow::Owned(name.to_lowercase())  // Needs modification, allocate
    } else {
        Cow::Borrowed(name)  // No modification needed, borrow
    }
}
```

---

## Unsafe Code Review (Most Critical!)

### Basic Requirements

```rust
// ❌ unsafe without safety docs — this is a red flag
unsafe fn bad_transmute<T, U>(t: T) -> U {
    std::mem::transmute(t)
}

// ✅ Every unsafe must explain: Why is it safe? What invariants?
/// Transmutes `T` to `U`.
///
/// # Safety
///
/// - `T` and `U` must have the same size and alignment
/// - `T` must be a valid bit pattern for `U`
/// - The caller ensures no references to `t` exist after this call
unsafe fn documented_transmute<T, U>(t: T) -> U {
    // SAFETY: Caller guarantees size/alignment match and bit validity
    std::mem::transmute(t)
}
```

### Unsafe Block Comments

```rust
// ❌ unsafe block without explanation
fn bad_get_unchecked(slice: &[u8], index: usize) -> u8 {
    unsafe { *slice.get_unchecked(index) }
}

// ✅ Every unsafe block must have SAFETY comment
fn good_get_unchecked(slice: &[u8], index: usize) -> u8 {
    debug_assert!(index < slice.len(), "index out of bounds");
    // SAFETY: We verified index < slice.len() via debug_assert.
    // In release builds, callers must ensure valid index.
    unsafe { *slice.get_unchecked(index) }
}

// ✅ Encapsulate unsafe to provide safe API
pub fn checked_get(slice: &[u8], index: usize) -> Option<u8> {
    if index < slice.len() {
        // SAFETY: bounds check performed above
        Some(unsafe { *slice.get_unchecked(index) })
    } else {
        None
    }
}
```

### Common Unsafe Patterns

```rust
// ✅ FFI boundary
extern "C" {
    fn external_function(ptr: *const u8, len: usize) -> i32;
}

pub fn safe_wrapper(data: &[u8]) -> Result<i32, Error> {
    // SAFETY: data.as_ptr() is valid for data.len() bytes,
    // and external_function only reads from the buffer.
    let result = unsafe {
        external_function(data.as_ptr(), data.len())
    };
    if result < 0 {
        Err(Error::from_code(result))
    } else {
        Ok(result)
    }
}

// ✅ Performance-critical path unsafe
pub fn fast_copy(src: &[u8], dst: &mut [u8]) {
    assert_eq!(src.len(), dst.len(), "slices must be equal length");
    // SAFETY: src and dst are valid slices of equal length,
    // and dst is mutable so no aliasing.
    unsafe {
        std::ptr::copy_nonoverlapping(
            src.as_ptr(),
            dst.as_mut_ptr(),
            src.len()
        );
    }
}
```

---

## Async Code

### Avoid Blocking Operations

```rust
// ❌ Blocking in async context — starves other tasks
async fn bad_async() {
    let data = std::fs::read_to_string("file.txt").unwrap();  // Blocking!
    std::thread::sleep(Duration::from_secs(1));  // Blocking!
}

// ✅ Use async API
async fn good_async() -> Result<String> {
    let data = tokio::fs::read_to_string("file.txt").await?;
    tokio::time::sleep(Duration::from_secs(1)).await;
    Ok(data)
}

// ✅ If blocking operation is necessary, use spawn_blocking
async fn with_blocking() -> Result<Data> {
    let result = tokio::task::spawn_blocking(|| {
        // Safe to do blocking operations here
        expensive_cpu_computation()
    }).await?;
    Ok(result)
}
```

### Mutex and .await

```rust
// ❌ Holding std::sync::Mutex across .await — may deadlock
async fn bad_lock(mutex: &std::sync::Mutex<Data>) {
    let guard = mutex.lock().unwrap();
    async_operation().await;  // Waiting while holding lock!
    process(&guard);
}

// ✅ Option 1: Minimize lock scope
async fn good_lock_scoped(mutex: &std::sync::Mutex<Data>) {
    let data = {
        let guard = mutex.lock().unwrap();
        guard.clone()  // Release lock immediately
    };
    async_operation().await;
    process(&data);
}

// ✅ Option 2: Use tokio::sync::Mutex (can cross await)
async fn good_lock_tokio(mutex: &tokio::sync::Mutex<Data>) {
    let guard = mutex.lock().await;
    async_operation().await;  // OK: tokio Mutex designed for crossing await
    process(&guard);
}

// 💡 Selection guide:
// - std::sync::Mutex: Low contention, short critical sections, no crossing await
// - tokio::sync::Mutex: Need to cross await, high contention scenarios
```

### Async Trait Methods

```rust
// ❌ async trait method pitfalls (older versions)
#[async_trait]
trait BadRepository {
    async fn find(&self, id: i64) -> Option<Entity>;  // Implicit Box
}

// ✅ Rust 1.75+: Native async trait methods
trait Repository {
    async fn find(&self, id: i64) -> Option<Entity>;

    // Return concrete Future type to avoid allocation
    fn find_many(&self, ids: &[i64]) -> impl Future<Output = Vec<Entity>> + Send;
}

// ✅ For scenarios needing dyn
trait DynRepository: Send + Sync {
    fn find(&self, id: i64) -> Pin<Box<dyn Future<Output = Option<Entity>> + Send + '_>>;
}
```

---

## Cancellation Safety

### What is Cancellation Safety

```rust
// When a Future is dropped at an .await point, what state is it in?
// A cancellation-safe Future: Can be safely cancelled at any await point
// A cancellation-unsafe Future: Cancellation may cause data loss or inconsistent state

// ❌ Example of cancellation-unsafe code
async fn cancel_unsafe(conn: &mut Connection) -> Result<()> {
    let data = receive_data().await;  // If cancelled here...
    conn.send_ack().await;  // ...ack is never sent, data may be lost
    Ok(())
}

// ✅ Cancellation-safe version
async fn cancel_safe(conn: &mut Connection) -> Result<()> {
    // Use transaction or atomic operation to ensure consistency
    let transaction = conn.begin_transaction().await?;
    let data = receive_data().await;
    transaction.commit_with_ack(data).await?;  // Atomic operation
    Ok(())
}
```

### Cancellation Safety in select!

```rust
use tokio::select;

// ❌ Using cancellation-unsafe Futures in select!
async fn bad_select(stream: &mut TcpStream) {
    let mut buffer = vec![0u8; 1024];
    loop {
        select! {
            // If timeout fires first, read is cancelled
            // Partially read data may be lost!
            result = stream.read(&mut buffer) => {
                handle_data(&buffer[..result?]);
            }
            _ = tokio::time::sleep(Duration::from_secs(5)) => {
                println!("Timeout");
            }
        }
    }
}

// ✅ Use cancellation-safe API
async fn good_select(stream: &mut TcpStream) {
    let mut buffer = vec![0u8; 1024];
    loop {
        select! {
            // tokio::io::AsyncReadExt::read is cancellation-safe
            // On cancellation, unread data stays in the stream
            result = stream.read(&mut buffer) => {
                match result {
                    Ok(0) => break,  // EOF
                    Ok(n) => handle_data(&buffer[..n]),
                    Err(e) => return Err(e),
                }
            }
            _ = tokio::time::sleep(Duration::from_secs(5)) => {
                println!("Timeout, retrying...");
            }
        }
    }
}

// ✅ Use tokio::pin! to ensure Futures can be safely reused
async fn pinned_select() {
    let sleep = tokio::time::sleep(Duration::from_secs(10));
    tokio::pin!(sleep);

    loop {
        select! {
            _ = &mut sleep => {
                println!("Timer elapsed");
                break;
            }
            data = receive_data() => {
                process(data).await;
                // sleep continues counting down, doesn't reset
            }
        }
    }
}
```

### Documenting Cancellation Safety

```rust
/// Reads a complete message from the stream.
///
/// # Cancel Safety
///
/// This method is **not** cancel safe. If cancelled while reading,
/// partial data may be lost and the stream state becomes undefined.
/// Use `read_message_cancel_safe` if cancellation is expected.
async fn read_message(stream: &mut TcpStream) -> Result<Message> {
    let len = stream.read_u32().await?;
    let mut buffer = vec![0u8; len as usize];
    stream.read_exact(&mut buffer).await?;
    Ok(Message::from_bytes(&buffer))
}

/// Reads a message with cancel safety.
///
/// # Cancel Safety
///
/// This method is cancel safe. If cancelled, any partial data
/// is preserved in the internal buffer for the next call.
async fn read_message_cancel_safe(reader: &mut BufferedReader) -> Result<Message> {
    reader.read_message_buffered().await
}
```

---

## spawn vs await

### When to Use spawn

```rust
// ❌ Unnecessary spawn — adds overhead, loses structured concurrency
async fn bad_unnecessary_spawn() {
    let handle = tokio::spawn(async {
        simple_operation().await
    });
    handle.await.unwrap();  // Why not just await directly?
}

// ✅ Direct await for simple operations
async fn good_direct_await() {
    simple_operation().await;
}

// ✅ spawn for true parallel execution
async fn good_parallel_spawn() {
    let task1 = tokio::spawn(fetch_from_service_a());
    let task2 = tokio::spawn(fetch_from_service_b());

    // Two requests executed in parallel
    let (result1, result2) = tokio::try_join!(task1, task2)?;
}

// ✅ spawn for background tasks (fire-and-forget)
async fn good_background_spawn() {
    // Start background task, don't wait for completion
    tokio::spawn(async {
        cleanup_old_sessions().await;
        log_metrics().await;
    });

    // Continue with other work
    handle_request().await;
}
```

### spawn's 'static Requirement

```rust
// ❌ spawn's Future must be 'static
async fn bad_spawn_borrow(data: &Data) {
    tokio::spawn(async {
        process(data).await;  // Error: `data` is not 'static
    });
}

// ✅ Option 1: Clone data
async fn good_spawn_clone(data: &Data) {
    let owned = data.clone();
    tokio::spawn(async move {
        process(&owned).await;
    });
}

// ✅ Option 2: Use Arc for sharing
async fn good_spawn_arc(data: Arc<Data>) {
    let data = Arc::clone(&data);
    tokio::spawn(async move {
        process(&data).await;
    });
}

// ✅ Option 3: Use scoped tasks (tokio-scoped or async-scoped)
async fn good_scoped_spawn(data: &Data) {
    // Assuming using async-scoped crate
    async_scoped::scope(|s| async {
        s.spawn(async {
            process(data).await;  // Can borrow
        });
    }).await;
}
```

### JoinHandle Error Handling

```rust
// ❌ Ignoring spawn errors
async fn bad_ignore_spawn_error() {
    let handle = tokio::spawn(async {
        risky_operation().await
    });
    let _ = handle.await;  // Ignored panic and errors
}

// ✅ Properly handle JoinHandle results
async fn good_handle_spawn_error() -> Result<()> {
    let handle = tokio::spawn(async {
        risky_operation().await
    });

    match handle.await {
        Ok(Ok(result)) => {
            // Task completed successfully
            process_result(result);
            Ok(())
        }
        Ok(Err(e)) => {
            // Task internal error
            Err(e.into())
        }
        Err(join_err) => {
            // Task panicked or was cancelled
            if join_err.is_panic() {
                error!("Task panicked: {:?}", join_err);
            }
            Err(anyhow!("Task failed: {}", join_err))
        }
    }
}
```

### Structured Concurrency vs spawn

```rust
// ✅ Prefer join! (structured concurrency)
async fn structured_concurrency() -> Result<(A, B, C)> {
    // All tasks within the same scope
    // If any fails, others are cancelled
    tokio::try_join!(
        fetch_a(),
        fetch_b(),
        fetch_c()
    )
}

// ✅ When using spawn, consider task lifecycle
struct TaskManager {
    handles: Vec<JoinHandle<()>>,
}

impl TaskManager {
    async fn shutdown(self) {
        // Graceful shutdown: wait for all tasks to complete
        for handle in self.handles {
            if let Err(e) = handle.await {
                error!("Task failed during shutdown: {}", e);
            }
        }
    }

    async fn abort_all(self) {
        // Force shutdown: cancel all tasks
        for handle in self.handles {
            handle.abort();
        }
    }
}
```

---

## Error Handling

### Library vs Application Error Types

```rust
// ❌ Library using anyhow — caller can't match errors
pub fn parse_config(s: &str) -> anyhow::Result<Config> { ... }

// ✅ Library uses thiserror, application uses anyhow
#[derive(Debug, thiserror::Error)]
pub enum ConfigError {
    #[error("invalid syntax at line {line}: {message}")]
    Syntax { line: usize, message: String },
    #[error("missing required field: {0}")]
    MissingField(String),
    #[error(transparent)]
    Io(#[from] std::io::Error),
}

pub fn parse_config(s: &str) -> Result<Config, ConfigError> { ... }
```

### Preserving Error Context

```rust
// ❌ Swallowing error context
fn bad_error() -> Result<()> {
    operation().map_err(|_| anyhow!("failed"))?;  // Original error lost
    Ok(())
}

// ✅ Use context to preserve error chain
fn good_error() -> Result<()> {
    operation().context("failed to perform operation")?;
    Ok(())
}

// ✅ Use with_context for lazy computation
fn good_error_lazy() -> Result<()> {
    operation()
        .with_context(|| format!("failed to process file: {}", filename))?;
    Ok(())
}
```

### Error Type Design

```rust
// ✅ Use #[source] to preserve error chain
#[derive(Debug, thiserror::Error)]
pub enum ServiceError {
    #[error("database error")]
    Database(#[source] sqlx::Error),

    #[error("network error: {message}")]
    Network {
        message: String,
        #[source]
        source: reqwest::Error,
    },

    #[error("validation failed: {0}")]
    Validation(String),
}

// ✅ Implement From for common conversions
impl From<sqlx::Error> for ServiceError {
    fn from(err: sqlx::Error) -> Self {
        ServiceError::Database(err)
    }
}
```

---

## Performance

### Avoid Unnecessary collect()

```rust
// ❌ Unnecessary collect — intermediate allocation
fn bad_sum(items: &[i32]) -> i32 {
    items.iter()
        .filter(|x| **x > 0)
        .collect::<Vec<_>>()  // Unnecessary!
        .iter()
        .sum()
}

// ✅ Lazy iteration
fn good_sum(items: &[i32]) -> i32 {
    items.iter().filter(|x| **x > 0).copied().sum()
}
```

### String Concatenation

```rust
// ❌ Repeated allocation in string concatenation loop
fn bad_concat(items: &[&str]) -> String {
    let mut s = String::new();
    for item in items {
        s = s + item;  // Re-allocates every time!
    }
    s
}

// ✅ Use join or with_capacity
fn good_concat(items: &[&str]) -> String {
    items.join("")
}

// ✅ Pre-allocate with with_capacity
fn good_concat_capacity(items: &[&str]) -> String {
    let total_len: usize = items.iter().map(|s| s.len()).sum();
    let mut result = String::with_capacity(total_len);
    for item in items {
        result.push_str(item);
    }
    result
}

// ✅ Use write! macro
use std::fmt::Write;

fn good_concat_write(items: &[&str]) -> String {
    let mut result = String::new();
    for item in items {
        write!(result, "{}", item).unwrap();
    }
    result
}
```

### Avoid Unnecessary Allocations

```rust
// ❌ Unnecessary Vec allocation
fn bad_check_any(items: &[Item]) -> bool {
    let filtered: Vec<_> = items.iter()
        .filter(|i| i.is_valid())
        .collect();
    !filtered.is_empty()
}

// ✅ Use iterator methods
fn good_check_any(items: &[Item]) -> bool {
    items.iter().any(|i| i.is_valid())
}

// ❌ String::from for static strings
fn bad_static() -> String {
    String::from("error message")  // Runtime allocation
}

// ✅ Return &'static str
fn good_static() -> &'static str {
    "error message"  // No allocation
}
```

---

## Trait Design

### Avoid Over-Abstraction

```rust
// ❌ Over-abstraction — not Java, don't need Interface for everything
trait Processor { fn process(&self); }
trait Handler { fn handle(&self); }
trait Manager { fn manage(&self); }  // Too many traits

// ✅ Only create traits when you need polymorphism
// Concrete types are usually simpler and faster
struct DataProcessor {
    config: Config,
}

impl DataProcessor {
    fn process(&self, data: &Data) -> Result<Output> {
        // Direct implementation
    }
}
```

### Trait Objects vs Generics

```rust
// ❌ Unnecessary trait objects (dynamic dispatch)
fn bad_process(handler: &dyn Handler) {
    handler.handle();  // Vtable call
}

// ✅ Use generics (static dispatch, can be inlined)
fn good_process<H: Handler>(handler: &H) {
    handler.handle();  // May be inlined
}

// ✅ Trait objects are appropriate for: heterogeneous collections
fn store_handlers(handlers: Vec<Box<dyn Handler>>) {
    // Need to store different types of handlers
}

// ✅ Use impl Trait for return types
fn create_handler() -> impl Handler {
    ConcreteHandler::new()
}
```

---

## Testing

### Test Organization

```rust
// ✅ Group related tests with clear naming
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_parse_valid_json() {
        // ...
    }

    #[test]
    fn test_parse_invalid_json_returns_error() {
        // ...
    }

    /// Edge case: empty input should return default, not panic
    #[test]
    fn test_parse_empty_input_returns_default() {
        // ...
    }
}
```

### Property-Based Testing

```rust
// ✅ Use proptest for property-based testing
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_roundtrip_serialize_deserialize(
        data in any::<MyStruct>()
    ) {
        let serialized = serde_json::to_string(&data).unwrap();
        let deserialized: MyStruct = serde_json::from_str(&serialized).unwrap();
        assert_eq!(data, deserialized);
    }
}
```

### Integration Tests

```rust
// ✅ Integration tests in tests/ directory
// tests/integration_test.rs

use tama_core::*;

#[test]
fn test_full_workflow() {
    let config = Config::default();
    let mut system = System::new(config);
    system.load_model("test-model").unwrap();
    let result = system.infer("test input").unwrap();
    assert!(!result.is_empty());
}
```

### Mock Testing

```rust
// ✅ Use mockall for mocking
use mockall::*;

mock! {
    pub trait Database {
        fn get_user(&self, id: i64) -> Option<User>;
        fn save_user(&self, user: &User) -> Result<()>;
    }
}

#[test]
fn test_user_service() {
    let mut mock_db = MockDatabase::new();
    mock_db
        .expect_get_user()
        .with(eq(42))
        .once()
        .returning(|id| Some(User { id, name: "Alice".into() }));

    let service = UserService::new(mock_db);
    let user = service.get_user(42).unwrap();
    assert_eq!(user.name, "Alice");
}
```

---

## Macros

### Hygiene

```rust
// ✅ Macro variables are hygienic — they don't leak
macro_rules! log_and_return {
    ($expr:expr) => {{
        let result = $expr;
        log::info!("Result: {:?}", result);
        result
    }};
}

// ✅ Use ident!() for identifier hygiene
macro_rules! create_struct {
    ($name:ident) => {
        struct $name {
            value: i32,
        }
    };
}
```

### Debugging Macros

```rust
// ✅ Use debug assertions for debug-only checks
macro_rules! debug_check {
    ($cond:expr, $msg:expr) => {
        debug_assert!($cond, $msg);
    };
}

// ✅ Use eprintln! for debugging (stripped in release)
fn process(data: &[u8]) {
    eprintln!("Processing {} bytes", data.len());
    // ...
}
```

---

## Common Anti-Patterns

### Overuse of unwrap()

```rust
// ❌ unwrap() in production code — panics on error
fn bad_process(path: &str) -> String {
    let data = std::fs::read_to_string(path).unwrap();
    data.to_uppercase()
}

// ✅ Use proper error handling
fn good_process(path: &str) -> Result<String, std::io::Error> {
    let data = std::fs::read_to_string(path)?;
    Ok(data.to_uppercase())
}

// ✅ Use expect() with a helpful message if unwrap is truly justified
fn good_expect() -> String {
    let config = CONFIG.as_ref().expect("CONFIG must be initialized");
    config.name.clone()
}
```

### String Overuse

```rust
// ❌ Returning owned Strings when &str would suffice
fn bad_get_name(user: &User) -> String {
    user.name.clone()
}

// ✅ Return references when possible
fn good_get_name(user: &User) -> &str {
    &user.name
}

// ✅ Use Cow when you sometimes need to allocate
fn maybe_transform(s: &str) -> Cow<str> {
    if s.contains(' ') {
        Cow::Owned(s.replace(' ', "_"))
    } else {
        Cow::Borrowed(s)
    }
}
```

### Mutex Overuse

```rust
// ❌ Using Mutex for everything
struct BadState {
    data: Mutex<HashMap<String, i32>>,
}

// ✅ Use RwLock for read-heavy workloads
use std::sync::RwLock;

struct GoodState {
    data: RwLock<HashMap<String, i32>>,
}

// ✅ Use atomic types for simple counters
use std::sync::atomic::{AtomicUsize, Ordering};

struct Counter {
    count: AtomicUsize,
}
```

### N+1 Queries

```rust
// ❌ Fetching related data in a loop
async fn bad_get_users_with_posts(db: &Db) -> Vec<UserWithPosts> {
    let users = db.get_all_users().await?;
    let mut result = Vec::new();
    for user in users {
        let posts = db.get_posts_for_user(user.id).await?;  // N queries!
        result.push(UserWithPosts { user, posts });
    }
    Ok(result)
}

// ✅ Batch fetch with JOIN or IN clause
async fn good_get_users_with_posts(db: &Db) -> Result<Vec<UserWithPosts>> {
    let users = db.get_all_users().await?;
    let user_ids: Vec<i64> = users.iter().map(|u| u.id).collect();
    let posts = db.get_posts_for_users(&user_ids).await?;  // 1 query!
    Ok(merge_users_and_posts(users, posts))
}
```

### Ignoring must_use

```rust
// ❌ Ignoring Result return values
fn bad_ignore_result() {
    some_function_that_returns_result();  // Result is ignored!
}

// ✅ Handle or explicitly ignore with _
fn good_handle_result() {
    match some_function_that_returns_result() {
        Ok(_) => {},
        Err(e) => log::error!("Error: {}", e),
    }
}

fn good_explicit_ignore() {
    let _ = some_function_that_returns_result();  // Explicitly ignored
}
```

---

## Rust Review Checklist

### What the Compiler Can't Catch

**Business logic correctness**
- [ ] Edge cases handled correctly
- [ ] State machine transitions complete
- [ ] Race conditions in concurrent scenarios

**API Design**
- [ ] Public API is hard to misuse
- [ ] Type signatures clearly express intent
- [ ] Error type granularity is appropriate

### Ownership and Borrowing

- [ ] clone() is intentional, documented with reason
- [ ] Does Arc<Mutex<T>> really need shared state?
- [ ] RefCell usage has valid justification
- [ ] Lifetimes not overly complex
- [ ] Consider using Cow to avoid unnecessary allocations

### Unsafe Code (Most Important)

- [ ] Every unsafe block has SAFETY comment
- [ ] unsafe fn has # Safety documentation section
- [ ] Explains why it's safe, not just what it does
- [ ] Lists invariants that must be maintained
- [ ] Unsafe boundaries are as small as possible
- [ ] Considered whether a safe alternative exists

### Async/Concurrency

- [ ] No blocking in async (std::fs, thread::sleep)
- [ ] No holding std::sync locks across .await
- [ ] Spawn'd tasks satisfy 'static
- [ ] Lock acquisition order is consistent
- [ ] Channel buffer sizes are reasonable

### Cancellation Safety

- [ ] Futures in select! are cancellation-safe
- [ ] Async function cancellation safety documented
- [ ] Cancellation doesn't cause data loss or inconsistent state
- [ ] Using tokio::pin! for Futures that need reuse
- [ ] For non-cancel-safe operations, using `cancel_safe_futures` crate or manual guards

### spawn vs await

- [ ] spawn only used when true parallelism is needed
- [ ] Simple operations use direct await, don't spawn
- [ ] spawn's JoinHandle results properly handled
- [ ] Considered task lifecycle and shutdown strategy
- [ ] Prefer join!/try_join! for structured concurrency

### Error Handling

- [ ] Library: thiserror for structured errors
- [ ] Application: anyhow + context
- [ ] No unwrap/expect in production code
- [ ] Error messages helpful for debugging
- [ ] must_use return values handled
- [ ] Using #[source] to preserve error chain

### Performance

- [ ] Avoid unnecessary collect()
- [ ] Large data passed by reference
- [ ] Strings use with_capacity or write!
- [ ] impl Trait vs Box<dyn Trait> choice appropriate
- [ ] Hot paths avoid allocations
- [ ] Consider using Cow to reduce clones

### Testing

- [ ] Unit tests for pure functions
- [ ] Integration tests for workflows
- [ ] Edge cases tested (empty, boundary, error)
- [ ] Property-based tests for roundtrip/invariants
- [ ] Mocks used appropriately (not overused)
- [ ] Tests are deterministic (no randomness)

### Macros

- [ ] Macro variables are hygienic
- [ ] Macros have doc comments explaining usage
- [ ] Debug assertions used for debug-only checks
- [ ] Macros don't leak identifiers

### Anti-Patterns

- [ ] No unwrap() in production code (use expect() with message if justified)
- [ ] No string overuse (use &str, Cow where appropriate)
- [ ] No mutex overuse (use RwLock, atomics where appropriate)
- [ ] No N+1 queries (batch fetches)
- [ ] No ignored must_use values

### Code Quality

- [ ] cargo clippy zero warnings
- [ ] cargo fmt applied
- [ ] Documentation comments complete
- [ ] Tests cover edge cases
- [ ] Public API has doc examples
