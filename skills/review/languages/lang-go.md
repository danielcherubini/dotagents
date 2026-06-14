# Go Code Review Guide

Code review checklist based on Go official guidelines, Effective Go, and community best practices.

## Quick Review Checklist

### Must-Check Items
- [ ] Are errors properly handled (not ignored, with context)?
- [ ] Do goroutines have exit mechanisms (avoid leaks)?
- [ ] Is context properly passed and cancelled?
- [ ] Is receiver type selection reasonable (value/pointer)?
- [ ] Is code formatted with `gofmt`?

### High-Frequency Issues
- [ ] Loop variable capture issue (Go < 1.22)
- [ ] Nil checks completeness
- [ ] Maps initialized before use
- [ ] Defer in loops
- [ ] Variable shadowing

---

## 1. Error Handling

### 1.1 Never Ignore Errors

```go
// ❌ Bad: Ignoring errors
result, _ := SomeFunction()

// ✅ Good: Handle errors
result, err := SomeFunction()
if err != nil {
    return fmt.Errorf("some function failed: %w", err)
}
```

### 1.2 Error Wrapping and Context

```go
// ❌ Bad: Losing context
if err != nil {
    return err
}

// ❌ Bad: Using %v loses error chain
if err != nil {
    return fmt.Errorf("failed: %v", err)
}

// ✅ Good: Use %w to preserve error chain
if err != nil {
    return fmt.Errorf("failed to process user %d: %w", userID, err)
}
```

### 1.3 Using errors.Is and errors.As

```go
// ❌ Bad: Direct comparison (won't handle wrapped errors)
if err == sql.ErrNoRows {
    // ...
}

// ✅ Good: Use errors.Is (supports error chain)
if errors.Is(err, sql.ErrNoRows) {
    return nil, ErrNotFound
}

// ✅ Good: Use errors.As to extract specific types
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    log.Printf("path error: %s", pathErr.Path)
}
```

### 1.4 Custom Error Types

```go
// ✅ Recommended: Define sentinel errors
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
)

// ✅ Recommended: Custom errors with context
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error on %s: %s", e.Field, e.Message)
}
```

### 1.5 Handle Errors Only Once

```go
// ❌ Bad: Both logging and returning (duplicate handling)
if err != nil {
    log.Printf("error: %v", err)
    return err
}

// ✅ Good: Only return, let caller decide
if err != nil {
    return fmt.Errorf("operation failed: %w", err)
}

// ✅ Or: Only log and handle (don't return)
if err != nil {
    log.Printf("non-critical error: %v", err)
    // Continue with fallback logic
}
```

---

## 2. Concurrency and Goroutines

### 2.1 Avoid Goroutine Leaks

```go
// ❌ Bad: Goroutine can never exit
func bad() {
    ch := make(chan int)
    go func() {
        val := <-ch // Blocks forever, nobody sends
        fmt.Println(val)
    }()
    // Function returns, goroutine leaks
}

// ✅ Good: Use context or done channel
func good(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case val := <-ch:
            fmt.Println(val)
        case <-ctx.Done():
            return // Graceful exit
        }
    }()
}
```

### 2.2 Channel Usage Rules

```go
// ❌ Bad: Sending to nil channel (permanent block)
var ch chan int
ch <- 1 // Permanent block

// ❌ Bad: Sending to closed channel (panic!)
close(ch)
ch <- 1 // panic!

// ✅ Good: Sender closes the channel
func producer(ch chan<- int) {
    defer close(ch) // Sender is responsible for closing
    for i := 0; i < 10; i++ {
        ch <- i
    }
}

// ✅ Good: Receiver detects closure
for val := range ch {
    process(val)
}
// Or
val, ok := <-ch
if !ok {
    // Channel is closed
}
```

### 2.3 Using sync.WaitGroup

```go
// ❌ Bad: Add inside goroutine
var wg sync.WaitGroup
for i := 0; i < 10; i++ {
    go func() {
        wg.Add(1) // Race condition!
        defer wg.Done()
        work()
    }()
}
wg.Wait()

// ✅ Good: Add before starting goroutine
var wg sync.WaitGroup
for i := 0; i < 10; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        work()
    }()
}
wg.Wait()
```

### 2.4 Avoid Capturing Loop Variables (Go < 1.22)

```go
// ❌ Bad (Go < 1.22): Capturing loop variable
for _, item := range items {
    go func() {
        process(item) // All goroutines may use the same item
    }()
}

// ✅ Good: Pass as parameter
for _, item := range items {
    go func(it Item) {
        process(it)
    }(item)
}

// ✅ Go 1.22+: Default behavior fixed, new variable per iteration
```

### 2.5 Worker Pool Pattern

```go
// ✅ Recommended: Limit concurrency
func processWithWorkerPool(ctx context.Context, items []Item, workers int) error {
    jobs := make(chan Item, len(items))
    results := make(chan error, len(items))

    // Start workers
    for w := 0; w < workers; w++ {
        go func() {
            for item := range jobs {
                results <- process(item)
            }
        }()
    }

    // Send tasks
    for _, item := range items {
        jobs <- item
    }
    close(jobs)

    // Collect results
    for range items {
        if err := <-results; err != nil {
            return err
        }
    }
    return nil
}
```

---

## 3. Context Usage

### 3.1 Context as First Parameter

```go
// ❌ Bad: context not first parameter
func Process(data []byte, ctx context.Context) error

// ❌ Bad: context stored in struct
type Service struct {
    ctx context.Context // Don't do this!
}

// ✅ Good: Context as first parameter, named ctx
func Process(ctx context.Context, data []byte) error
```

### 3.2 Propagate, Don't Create New Root Contexts

```go
// ❌ Bad: Creating new root context in call chain
func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := context.Background() // Lost the request's context!
        process(ctx)
        next.ServeHTTP(w, r)
    })
}

// ✅ Good: Get and propagate from request
func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        ctx = context.WithValue(ctx, key, value)
        process(ctx)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### 3.3 Always Call cancel Function

```go
// ❌ Bad: Not calling cancel
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
// Missing cancel() call, may leak resources

// ✅ Good: Use defer to ensure call
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
defer cancel() // Call even on timeout
```

### 3.4 Respond to Context Cancellation

```go
// ✅ Recommended: Check context in long-running operations
func LongRunningTask(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err() // Return context.Canceled or context.DeadlineExceeded
        default:
            // Do a small chunk of work
            if err := doChunk(); err != nil {
                return err
            }
        }
    }
}
```

### 3.5 Distinguish Cancellation Reasons

```go
// ✅ Distinguish cancellation reasons by ctx.Err()
if err := ctx.Err(); err != nil {
    switch {
    case errors.Is(err, context.Canceled):
        log.Println("operation was canceled")
    case errors.Is(err, context.DeadlineExceeded):
        log.Println("operation timed out")
    }
    return err
}
```

---

## 4. Interface Design

### 4.1 Accept Interfaces, Return Structs

```go
// ❌ Not recommended: Accept concrete type
func SaveUser(db *sql.DB, user User) error

// ✅ Recommended: Accept interface (decoupled, easy to test)
type UserStore interface {
    Save(ctx context.Context, user User) error
}

func SaveUser(store UserStore, user User) error

// ❌ Not recommended: Return interface
func NewUserService() UserServiceInterface

// ✅ Recommended: Return concrete type
func NewUserService(store UserStore) *UserService
```

### 4.2 Define Interfaces at the Consumer

```go
// ❌ Not recommended: Define interface in implementation package
// package database
type Database interface {
    Query(ctx context.Context, query string) ([]Row, error)
    // ... 20 methods
}

// ✅ Recommended: Define minimal interface needed by consumer package
// package userservice
type UserQuerier interface {
    QueryUsers(ctx context.Context, filter Filter) ([]User, error)
}
```

### 4.3 Keep Interfaces Small and Focused

```go
// ❌ Not recommended: Large, all-purpose interface
type Repository interface {
    GetUser(id int) (*User, error)
    CreateUser(u *User) error
    UpdateUser(u *User) error
    DeleteUser(id int) error
    GetOrder(id int) (*Order, error)
    CreateOrder(o *Order) error
    // ... more methods
}

// ✅ Recommended: Small, focused interfaces
type UserReader interface {
    GetUser(ctx context.Context, id int) (*User, error)
}

type UserWriter interface {
    CreateUser(ctx context.Context, u *User) error
    UpdateUser(ctx context.Context, u *User) error
}

// Compose interfaces
type UserRepository interface {
    UserReader
    UserWriter
}
```

### 4.4 Avoid Excessive use of Empty Interfaces

```go
// ❌ Not recommended: Overusing interface{}
func Process(data interface{}) interface{}

// ✅ Recommended: Use generics (Go 1.18+)
func Process[T any](data T) T

// ✅ Recommended: Define concrete interface
type Processor interface {
    Process() Result
}
```

---

## 5. Receiver Type Selection

### 5.1 When to Use Pointer Receivers

```go
// ✅ Need to modify receiver
func (u *User) SetName(name string) {
    u.Name = name
}

// ✅ Receiver contains sync.Mutex or other sync primitives
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

// ✅ Receiver is a large struct (avoid copy overhead)
type LargeStruct struct {
    Data [1024]byte
    // ...
}

func (l *LargeStruct) Process() { /* ... */ }
```

### 5.2 When to Use Value Receivers

```go
// ✅ Small immutable struct receiver
type Point struct {
    X, Y float64
}

func (p Point) Distance(other Point) float64 {
    return math.Sqrt(math.Pow(p.X-other.X, 2) + math.Pow(p.Y-other.Y, 2))
}

// ✅ Alias of basic type
type Counter int

func (c Counter) String() string {
    return fmt.Sprintf("%d", c)
}

// ✅ Receiver is map, func, chan (itself a reference type)
type StringSet map[string]struct{}

func (s StringSet) Contains(key string) bool {
    _, ok := s[key]
    return ok
}
```

### 5.3 Consistency Principle

```go
// ❌ Not recommended: Mixed receiver types
func (u User) GetName() string   // Value receiver
func (u *User) SetName(n string) // Pointer receiver

// ✅ Recommended: If any method needs pointer receiver, use pointers for all
func (u *User) GetName() string { return u.Name }
func (u *User) SetName(n string) { u.Name = n }
```

---

## 6. Performance Optimization

### 6.1 Pre-allocate Slices

```go
// ❌ Not recommended: Dynamic growth
var result []int
for i := 0; i < 10000; i++ {
    result = append(result, i) // Multiple allocations and copies
}

// ✅ Recommended: Pre-allocate known size
result := make([]int, 0, 10000)
for i := 0; i < 10000; i++ {
    result = append(result, i)
}

// ✅ Or initialize directly
result := make([]int, 10000)
for i := 0; i < 10000; i++ {
    result[i] = i
}
```

### 6.2 Avoid Unnecessary Heap Allocations

```go
// ❌ May escape to heap
func NewUser() *User {
    return &User{} // Escapes to heap
}

// ✅ Consider returning value (if applicable)
func NewUser() User {
    return User{} // May allocate on stack
}

// Check escape analysis
// go build -gcflags '-m -m' ./...
```

### 6.3 Use sync.Pool for Object Reuse

```go
// ✅ Recommended: Use sync.Pool for frequently created/destroyed objects
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func ProcessData(data []byte) string {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufferPool.Put(buf)
    }()

    buf.Write(data)
    return buf.String()
}
```

### 6.4 String Concatenation Optimization

```go
// ❌ Not recommended: Using + in loop
var result string
for _, s := range strings {
    result += s // Creates new string each time
}

// ✅ Recommended: Use strings.Builder
var builder strings.Builder
for _, s := range strings {
    builder.WriteString(s)
}
result := builder.String()

// ✅ Or use strings.Join
result := strings.Join(strings, "")
```

### 6.5 Avoid interface{} Conversion Overhead

```go
// ❌ Using interface{} in hot path
func process(data interface{}) {
    switch v := data.(type) { // Type assertion has overhead
    case int:
        // ...
    }
}

// ✅ Use generics or concrete types in hot path
func process[T int | int64 | float64](data T) {
    // Type determined at compile time, no runtime overhead
}
```

---

## 7. Testing

### 7.1 Table-Driven Tests

```go
// ✅ Recommended: Table-driven tests
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive numbers", 1, 2, 3},
        {"with zero", 0, 5, 5},
        {"negative numbers", -1, -2, -3},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("Add(%d, %d) = %d; want %d",
                    tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```

### 7.2 Parallel Tests

```go
// ✅ Recommended: Independent test cases run in parallel
func TestParallel(t *testing.T) {
    tests := []struct {
        name  string
        input string
    }{
        {"test1", "input1"},
        {"test2", "input2"},
    }

    for _, tt := range tests {
        tt := tt // Go < 1.22 needs copy
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // Mark as parallelizable
            result := Process(tt.input)
            // assertions...
        })
    }
}
```

### 7.3 Using Interfaces for Mocking

```go
// ✅ Define interface for testing
type EmailSender interface {
    Send(to, subject, body string) error
}

// Production implementation
type SMTPSender struct { /* ... */ }

// Test Mock
type MockEmailSender struct {
    SendFunc func(to, subject, body string) error
}

func (m *MockEmailSender) Send(to, subject, body string) error {
    return m.SendFunc(to, subject, body)
}

func TestUserRegistration(t *testing.T) {
    mock := &MockEmailSender{
        SendFunc: func(to, subject, body string) error {
            if to != "test@example.com" {
                t.Errorf("unexpected recipient: %s", to)
            }
            return nil
        },
    }

    service := NewUserService(mock)
    // test...
}
```

### 7.4 Test Helper Functions

```go
// ✅ Use t.Helper() to mark helper functions
func assertEqual(t *testing.T, got, want interface{}) {
    t.Helper() // Error report shows caller location
    if got != want {
        t.Errorf("got %v, want %v", got, want)
    }
}

// ✅ Use t.Cleanup() to clean up resources
func TestWithTempFile(t *testing.T) {
    f, err := os.CreateTemp("", "test")
    if err != nil {
        t.Fatal(err)
    }
    t.Cleanup(func() {
        os.Remove(f.Name())
    })
    // test...
}
```

---

## 8. Common Pitfalls

### 8.1 Nil Slice vs Empty Slice

```go
var nilSlice []int     // nil, len=0, cap=0
emptySlice := []int{}  // not nil, len=0, cap=0
made := make([]int, 0) // not nil, len=0, cap=0

// ✅ JSON encoding difference
json.Marshal(nilSlice)   // null
json.Marshal(emptySlice) // []

// ✅ Recommended: Explicitly initialize when you need empty array JSON
if slice == nil {
    slice = []int{}
}
```

### 8.2 Map Initialization

```go
// ❌ Bad: Uninitialized map
var m map[string]int
m["key"] = 1 // panic: assignment to entry in nil map

// ✅ Good: Use make to initialize
m := make(map[string]int)
m["key"] = 1

// ✅ Or use literal
m := map[string]int{}
```

### 8.3 Defer in Loops

```go
// ❌ Potential issue: defer executes at function end
func processFiles(files []string) error {
    for _, file := range files {
        f, err := os.Open(file)
        if err != nil {
            return err
        }
        defer f.Close() // All files closed only when function ends!
        // process...
    }
    return nil
}

// ✅ Good: Use closure or extract function
func processFiles(files []string) error {
    for _, file := range files {
        if err := processFile(file); err != nil {
            return err
        }
    }
    return nil
}

func processFile(file string) error {
    f, err := os.Open(file)
    if err != nil {
        return err
    }
    defer f.Close()
    // process...
    return nil
}
```

### 8.4 Slice Underlying Array Sharing

```go
// ❌ Potential issue: Slices share underlying array
original := []int{1, 2, 3, 4, 5}
slice := original[1:3] // [2, 3]
slice[0] = 100         // Modifies original!
// original becomes [1, 100, 3, 4, 5]

// ✅ Good: Explicitly copy when you need an independent copy
slice := make([]int, 2)
copy(slice, original[1:3])
slice[0] = 100 // Doesn't affect original
```

### 8.5 String Substring Memory Leak

```go
// ❌ Potential issue: Substring holds entire underlying array
func getPrefix(s string) string {
    return s[:10] // Still references entire s's underlying array
}

// ✅ Good: Create independent copy (Go 1.18+)
func getPrefix(s string) string {
    return strings.Clone(s[:10])
}

// ✅ Go 1.18 and earlier
func getPrefix(s string) string {
    return string([]byte(s[:10]))
}
```

### 8.6 Interface Nil Trap

```go
// ❌ Trap: interface nil check
type MyError struct{}
func (e *MyError) Error() string { return "error" }

func returnsError() error {
    var e *MyError = nil
    return e // Returned error is not nil!
}

func main() {
    err := returnsError()
    if err != nil { // true! interface{type: *MyError, value: nil}
        fmt.Println("error:", err)
    }
}

// ✅ Good: Explicitly return nil
func returnsError() error {
    var e *MyError = nil
    if e == nil {
        return nil // Explicit nil return
    }
    return e
}
```

### 8.7 Time Comparison

```go
// ❌ Not recommended: Using == to compare time.Time directly
if t1 == t2 { // May fail due to monotonic clock difference
    // ...
}

// ✅ Recommended: Use Equal method
if t1.Equal(t2) {
    // ...
}

// ✅ Compare time ranges
if t1.Before(t2) || t1.After(t2) {
    // ...
}
```

---

## 9. Code Organization

### 9.1 Package Naming

```go
// ❌ Not recommended
package common   // Too broad
package utils    // Too broad
package helpers  // Too broad
package models   // Grouped by type

// ✅ Recommended: Name by functionality
package user     // User-related functionality
package order    // Order-related functionality
package postgres // PostgreSQL implementation
```

### 9.2 Avoid Circular Dependencies

```go
// ❌ Circular dependency
// package a imports package b
// package b imports package a

// ✅ Solution 1: Extract shared types to independent package
// package types (shared types)
// package a imports types
// package b imports types

// ✅ Solution 2: Decouple with interfaces
// package a defines interface
// package b implements interface
```

### 9.3 Exported Identifier Conventions

```go
// ✅ Only export necessary identifiers
type UserService struct {
    db *sql.DB // Private
}

func (s *UserService) GetUser(id int) (*User, error) // Public
func (s *UserService) validate(u *User) error         // Private

// ✅ Internal package limits access
// internal/database/... can only be imported by same project code
```

---

## 10. Tools and Checks

### 10.1 Essential Tools

```bash
# Formatting (required)
gofmt -w .
goimports -w .

# Static analysis
go vet ./...

# Race detection
go test -race ./...

# Escape analysis
go build -gcflags '-m -m' ./...
```

### 10.2 Recommended Linters

```bash
# golangci-lint (integrates multiple linters)
golangci-lint run

# Common checks
# - errcheck: Check unhandled errors
# - gosec: Security checks
# - ineffassign: Ineffective assignments
# - staticcheck: Static analysis
# - unused: Unused code
```

### 10.3 Benchmark Tests

```go
// ✅ Performance benchmark tests
func BenchmarkProcess(b *testing.B) {
    data := prepareData()
    b.ResetTimer() // Reset timer

    for i := 0; i < b.N; i++ {
        Process(data)
    }
}

# Run benchmark
# go test -bench=. -benchmem ./...
```

---

## Go 1.24 New Features

### Generic Type Aliases
```go
// ✅ Go 1.24: Fully supports generic type aliases
type MyMap[K, V] = map[K]V  // Now fully supported
type StringIntMap = MyMap[string, int]
```

### JSON `omitzero` Tag (Go 1.24)
```go
// ✅ Go 1.24: New omitzero tag for JSON marshaling
// Unlike omitempty (which omits empty values), omitzero omits zero values
type Config struct {
    Port int `json:"port,omitempty,omitzero"`  // Omitted if 0
}
```

### Tool Directives
```go
// ✅ Go 1.24: Track executable dependencies
go tool go.mod
require example.com/cli-tool v1.0.0 // tool
```

## Reference Resources

- [Effective Go](https://go.dev/doc/effective_go)
- [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments)
- [Go Common Mistakes](https://go.dev/wiki/CommonMistakes)
- [100 Go Mistakes](https://100go.co/)
- [Go Proverbs](https://go-proverbs.github.io/)
- [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
