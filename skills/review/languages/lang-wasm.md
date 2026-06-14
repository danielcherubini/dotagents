# WASM/Frontend Code Review Guide

> WASM and frontend code review guide. Focus on memory management, JS interop, bundle size, accessibility, and browser compatibility.

## Table of Contents

- [Memory Management](#memory-management)
- [JS Interop (wasm-bindgen)](#js-interop-wasm-bindgen)
- [Bundle Size Optimization](#bundle-size-optimization)
- [Accessibility (a11y)](#accessibility-a11y)
- [Browser Compatibility](#browser-compatibility)
- [State Management](#state-management)
- [Error Handling](#error-handling)
- [Testing](#testing)
- [Security](#security)
- [Review Checklist](#wasm-review-checklist)

---

## Memory Management

### Avoid Unnecessary Allocations

```rust
// ❌ Allocating strings unnecessarily in hot paths
fn bad_format_name(first: &str, last: &str) -> String {
    let full = format!("{} {}", first, last);  // Allocation
    full.to_uppercase()  // Another allocation
}

// ✅ Use Cow to avoid unnecessary allocations
use std::borrow::Cow;

fn good_format_name(first: &str, last: &str) -> Cow<str> {
    let full = format!("{} {}", first, last);
    if full.is_ascii() {
        Cow::Owned(full.to_uppercase())  // Only allocate when needed
    } else {
        Cow::Borrowed(full.as_str())  // Borrow if already uppercase-compatible
    }
}
```

### Watch for Memory Leaks in WASM

```rust
// ❌ Closure capturing large structs by value
fn bad_closure(data: LargeStruct) {
    Closure::wrap(Box::new(move || {
        // data is captured by value — large struct copied into closure
        process(data);
    }));
}

// ✅ Capture references or use Arc for shared ownership
fn good_closure(data: Arc<LargeStruct>) {
    Closure::wrap(Box::new(move || {
        // Arc clone is cheap (just incrementing reference count)
        process(&*data);
    }));
}
```

### WASM Memory Growth

```rust
// ❌ Unbounded growth — no cap on memory
// WASM memory grows with each allocation, never shrinks

// ✅ Set memory limits in Cargo.toml
// [package.metadata.wasm-pack.profile.release]
// wasm-opt = ['-Oz']  // Optimize for size

// ✅ Use wasm-opt to reduce binary size
// wasm-opt -Oz -o output.wasm input.wasm
```

---

## JS Interop (wasm-bindgen)

### Avoid Unnecessary JS FFI Calls

```rust
// ❌ Calling JS in a hot loop
fn bad_process(items: &[Item]) {
    for item in items {
        // Each call has overhead
        web_sys::console::log_1(&JsValue::from_str(&item.to_string()));
    }
}

// ✅ Batch JS calls
fn good_process(items: &[Item]) {
    let batch: String = items.iter()
        .map(|i| i.to_string())
        .collect::<Vec<_>>()
        .join(", ");
    web_sys::console::log_1(&JsValue::from_str(&batch));
}
```

### Proper Cleanup of Closures

```rust
// ❌ Forgetting to forget — closure memory leaked
fn bad_cleanup() {
    let closure = Closure::wrap(Box::new(move || {
        // ...
    }));
    window.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref());
    // closure is dropped here but event listener still holds reference
}

// ✅ Forget the closure to transfer ownership to JS
fn good_cleanup() {
    let closure = Closure::wrap(Box::new(move || {
        // ...
    }));
    window.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref());
    closure.forget();  // Transfer ownership to JS
}
```

### Type Safety in JS Interop

```rust
// ❌ Using JsValue for everything — loses type safety
fn bad_js_interop(value: JsValue) {
    let num = value.as_f64().unwrap();  // Panics if not a number
}

// ✅ Use typed bindings when possible
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern "C" {
    fn jsFunction(arg: u32) -> JsValue;
}

fn good_js_interop(arg: u32) -> Result<u32, JsValue> {
    let result = jsFunction(arg);
    result.as_f64()
        .map(|v| v as u32)
        .ok_or_else(|| JsValue::from_str("expected number"))
}
```

---

## Bundle Size Optimization

### Tree Shaking

```rust
// ❌ Importing everything
use leptos::*;  // Imports ALL of leptos

// ✅ Import only what you need
use leptos::prelude::*;  // Only the prelude
use leptos::component;   // Only what you use
```

### Conditional Compilation

```rust
// ✅ Exclude debug code from release builds
#[cfg(debug_assertions)]
fn debug_log(msg: &str) {
    web_sys::console::log_1(&JsValue::from_str(msg));
}

#[cfg(not(debug_assertions))]
fn debug_log(_: &str) {}  // No-op in release

// ✅ Use feature flags to exclude large dependencies
#[cfg(feature = "analytics")]
use analytics::track_event;

#[cfg(not(feature = "analytics"))]
fn track_event(_: &str) {}
```

### Webpack/Vite Configuration

```javascript
// ✅ Optimize bundle size
export default {
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          utils: ['lodash', 'date-fns'],
        },
      },
    },
  },
};
```

---

## Accessibility (a11y)

### ARIA Labels

```rust
// ❌ No accessibility information
view! {
    <button on:click=move |_| { save() }>
        <span class="icon">"💾"</span>
    </button>
}

// ✅ Add ARIA labels for screen readers
view! {
    <button
        on:click=move |_| { save() }
        attr:aria-label="Save changes"
        attr:title="Save changes"
    >
        <span class="icon">"💾"</span>
    </button>
}
```

### Keyboard Navigation

```rust
// ✅ Ensure all interactive elements are keyboard accessible
view! {
    <div
        class="custom-dropdown"
        role="button"
        tabindex="0"
        on:keydown=move |e| {
            if e.key() == "Enter" || e.key() == " " {
                e.prevent_default();
                toggle();
            }
        }
        on:click=toggle
    >
        // ...
    </div>
}
```

### Focus Management

```rust
// ✅ Manage focus after navigation or modal open/close
fn open_modal() {
    // Focus the first interactive element
    if let Some(el) = web_sys::window()
        .and_then(|w| w.document())
        .and_then(|d| d.get_element_by_id("modal-first-input"))
    {
        el.focus();
    }
}

fn close_modal() {
    // Return focus to the trigger element
    if let Some(el) = web_sys::window()
        .and_then(|w| w.document())
        .and_then(|d| d.get_element_by_id("modal-trigger"))
    {
        el.focus();
    }
}
```

---

## Browser Compatibility

### Feature Detection

```rust
// ❌ Assuming API availability
fn bad_feature_usage() {
    // This will crash in older browsers
    let storage = web_sys::window().unwrap().local_storage().unwrap();
}

// ✅ Feature detection with fallback
fn good_feature_usage() {
    let storage = web_sys::window()
        .and_then(|w| w.local_storage().ok())
        .flatten()
        .or_else(|| {
            // Fallback: use in-memory storage
            Some(MemoryStorage::new())
        });
}
```

### Polyfill Strategy

```javascript
// ✅ Load polyfills conditionally
if (!Array.prototype.at) {
    require('core-js/features/array/at');
}

// ✅ Use browserslist for automatic polyfilling
// browserslist: "> 0.5%, last 2 versions, not dead, not IE 11"
```

---

## State Management

### Signal/Reactivity Patterns

```rust
// ❌ Using RwSignal when Signal would suffice
let count = RwSignal::new(0);  // Overkill if only reading

// ✅ Use Signal for read-only access
let count = create_signal(0);
let (get_count, _) = count;

// ✅ Use RwSignal when you need to write
let count = create_rw_signal(0);
```

### Derived Signals

```rust
// ❌ Computing derived values in the view (recomputes on every render)
view! {
    <p>{format!("Total: {}", items.iter().map(|i| i.price).sum::<f64>())}</p>
}

// ✅ Use Signal::derive for cached derived values
let total = Signal::derive(move || {
    items.get().iter().map(|i| i.price).sum::<f64>()
});
view! {
    <p>{format!("Total: {}", total.get())}</p>
}
```

### Effects vs Computed

```rust
// ❌ Using Effect for computed values
Effect::new(move |_| {
    let total = items.iter().map(|i| i.price).sum::<f64>();
    // total is recomputed every time any dependency changes
    display_total.set(total);
});

// ✅ Use Memo for cached computed values
let total = Memo::new(move |_| {
    items.get().iter().map(|i| i.price).sum::<f64>()
});
```

---

## Error Handling

### WASM Error Boundaries

```rust
// ❌ Unhandled errors crash the entire app
view! {
    <Suspense fallback=|| view! { "Loading..." }>
        {move || {
            data.get().map(|d| render_data(d)).unwrap_or_default()
        }}
    </Suspense>
}

// ✅ Error boundary with graceful fallback
view! {
    <ErrorBoundary fallback=|errors| {
        view! {
            <div class="error-boundary">
                <p>"Something went wrong"</p>
                <button on:click=move |_| errors.set(vec![])>"Retry"</button>
            </div>
        }
    }>
        {move || {
            data.get().map(|d| render_data(d)).unwrap_or_default()
        }}
    </ErrorBoundary>
}
```

### Async Error Handling

```rust
// ✅ Handle async errors gracefully
let resource = Resource::new(
    move || refresh.get(),
    move |key: u32| async move {
        match fetch_data(key).await {
            Ok(data) => Ok(data),
            Err(e) => {
                // Log error but don't crash
                web_sys::console::error_1(&JsValue::from_str(&format!("Fetch error: {}", e)));
                Err(e)
            }
        }
    }
);
```

---

## Testing

### Unit Testing WASM

```rust
// ✅ Test pure logic on host target
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_format_price() {
        assert_eq!(format_price(1234), "$12.34");
        assert_eq!(format_price(0), "$0.00");
        assert_eq!(format_price(100000), "$1,000.00");
    }
}

// ✅ Test WASM-specific code with wasm-bindgen-test
#[cfg(test)]
#[wasm_bindgen_test]
mod wasm_tests {
    use wasm_bindgen_test::*;

    #[wasm_bindgen_test]
    fn test_js_interop() {
        // Test JS interop in WASM environment
        let result = call_js_function(42);
        assert_eq!(result.as_f64(), Some(84.0));
    }
}
```

### Integration Testing

```rust
// ✅ Test component rendering
#[cfg(test)]
mod integration_tests {
    use leptos::prelude::*;

    #[test]
    fn test_component_render() {
        let (view, _) = leptos::view! {
            <MyComponent value=42 />
        };
        // Assert rendered output
    }
}
```

---

## Security

### XSS Prevention

```rust
// ✅ Leptos escapes by default in view! macro
view! {
    <p>{user_input}</p>  // Automatically escaped
}

// ❌ Using set_inner_html — bypasses escaping
view! {
    <p set:inner_html=user_input />  // DANGEROUS: user_input must be sanitized
}
```

### CSRF Protection

```rust
// ✅ Use CSRF tokens for state-changing operations
pub fn post_request(url: &str) -> RequestBuilder {
    let mut builder = Request::post(url);
    if let Some(token) = get_csrf_token() {
        builder = builder.header("X-CSRF-Token", &token);
    }
    builder
}
```

### Content Security Policy

```javascript
// ✅ Set CSP headers
const csp = [
    "default-src 'self'",
    "script-src 'self' 'unsafe-inline'",
    "style-src 'self' 'unsafe-inline'",
    "img-src 'self' data: https:",
    "connect-src 'self' https://api.example.com",
].join('; ');

response.headers.set('Content-Security-Policy', csp);
```

---

## WASM Review Checklist

### Memory & Performance
- [ ] No unnecessary allocations in hot paths
- [ ] Closures don't capture large structs by value
- [ ] WASM binary size is reasonable (< 500KB for most apps)
- [ ] wasm-opt is used in release builds
- [ ] No memory leaks from forgotten closures

### JS Interop
- [ ] JS FFI calls are minimized (batched where possible)
- [ ] Closures are properly forgotten when transferred to JS
- [ ] Type-safe bindings used instead of raw JsValue where possible
- [ ] Error handling for JS interop failures

### Bundle Size
- [ ] Only necessary dependencies are imported
- [ ] Tree shaking is effective (no dead code)
- [ ] Large dependencies are chunked separately
- [ ] Feature flags exclude non-essential code

### Accessibility
- [ ] All interactive elements have ARIA labels
- [ ] Keyboard navigation works for all interactive elements
- [ ] Focus is managed correctly (modals, navigation)
- [ ] Color contrast meets WCAG AA standards
- [ ] Screen reader testable

### Browser Compatibility
- [ ] Feature detection used instead of assumptions
- [ ] Polyfills loaded for missing features
- [ ] browserslist configured appropriately
- [ ] Tested on target browsers

### State Management
- [ ] Signal vs RwSignal chosen correctly
- [ ] Derived values use Signal::derive or Memo
- [ ] Effects don't cause unnecessary re-renders
- [ ] Resource loading patterns are efficient

### Error Handling
- [ ] Error boundaries catch rendering errors
- [ ] Async errors are handled gracefully
- [ ] User-friendly error messages
- [ ] Error logging for debugging

### Security
- [ ] No set_inner_html with unsanitized input
- [ ] CSRF tokens used for state-changing operations
- [ ] CSP headers configured
- [ ] No sensitive data in client-side code
- [ ] Dependencies are up to date (no known vulnerabilities)

### Testing
- [ ] Pure logic tested on host target
- [ ] WASM-specific code tested with wasm-bindgen-test
- [ ] Component rendering tested
- [ ] Integration tests for critical paths

---

## Common WASM Pitfalls

| Pitfall | Solution |
|---------|----------|
| Large bundle size | Use wasm-opt, tree shaking, chunking |
| Memory leaks | Forget closures, avoid capturing large values |
| Slow JS interop | Batch calls, use typed bindings |
| No accessibility | Add ARIA labels, keyboard navigation |
| Browser crashes | Feature detection, error boundaries |
| No error handling | Handle async errors, provide fallbacks |
| Unoptimized builds | Use wasm-opt, cfg(feature = "release") |
| Missing tests | Use wasm-bindgen-test for WASM-specific tests |
