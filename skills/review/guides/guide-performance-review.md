# Performance Review Guide

Performance review guide covering frontend, backend, database, algorithm complexity, and API performance.

## Table of Contents

- [Frontend Performance (Core Web Vitals)](#frontend-performance-core-web-vitals)
- [JavaScript Performance](#javascript-performance)
- [Memory Management](#memory-management)
- [Database Performance](#database-performance)
- [API Performance](#api-performance)
- [Algorithm Complexity](#algorithm-complexity)
- [Performance Review Checklist](#performance-review-checklist)

---

## Frontend Performance (Core Web Vitals)

### Core Web Vitals (Current)

| Metric | Full Name | Good | Needs Improvement | Poor |
|--------|-----------|------|-------------------|------|
| **LCP** | Largest Contentful Paint | ≤ 2.5s | 2.5-4.0s | > 4.0s |
| **INP** | Interaction to Next Paint | ≤ 200ms | 200-500ms | > 500ms |
| **CLS** | Cumulative Layout Shift | ≤ 0.1 | 0.1-0.25 | > 0.25 |

### Additional Metrics to Monitor

| Metric | Good | Needs Improvement | Poor |
|--------|------|-------------------|------|
| **FCP** | First Contentful Paint | ≤ 1.8s | 1.8-3.0s | > 3.0s |
| **TBT** | Total Blocking Time | ≤ 200ms | 200-600ms | > 600ms |
| **TTI** | Time to Interactive | ≤ 3.8s | 3.8-7.3s | > 7.3s |

### LCP Optimization Checklist

```javascript
// ❌ Lazy-loaded LCP image - delays critical content
<img src="hero.jpg" loading="lazy" />

// ✅ Load LCP image immediately
<img src="hero.jpg" fetchpriority="high" />

// ❌ Unoptimized image format
<img src="hero.png" />  // PNG too large

// ✅ Modern image formats + responsive
<picture>
  <source srcset="hero.avif" type="image/avif" />
  <source srcset="hero.webp" type="image/webp" />
  <img src="hero.jpg" alt="Hero" />
</picture>
```

**Review points:**
- [ ] Does the LCP element have `fetchpriority="high"`?
- [ ] Are WebP/AVIF formats used?
- [ ] Is there server-side rendering or static generation?
- [ ] Is CDN configured correctly?

### FCP Optimization Checklist

```html
<!-- ❌ Render-blocking CSS -->
<link rel="stylesheet" href="all-styles.css" />

<!-- ✅ Inline critical CSS + load rest asynchronously -->
<style>/* Critical styles for first screen */</style>
<link rel="preload" href="styles.css" as="style" onload="this.onload=null;this.rel='stylesheet'" />

<!-- ❌ Render-blocking font -->
@font-face {
  font-family: 'CustomFont';
  src: url('font.woff2');
}

<!-- ✅ Font display optimization -->
@font-face {
  font-family: 'CustomFont';
  src: url('font.woff2');
  font-display: swap;  /* Show system font first, switch after loading */
}
```

### INP Optimization Checklist

```javascript
// ❌ Long task blocking the main thread
button.addEventListener('click', () => {
  // Synchronous operation taking 500ms
  processLargeData(data);
  updateUI();
});

// ✅ Split long tasks
button.addEventListener('click', async () => {
  // Yield to the main thread
  await scheduler.yield?.() ?? new Promise(r => setTimeout(r, 0));

  // Process in chunks
  for (const chunk of chunks) {
    processChunk(chunk);
    await scheduler.yield?.();
  }
  updateUI();
});

// ✅ Use Web Worker for complex computation
const worker = new Worker('heavy-computation.js');
worker.postMessage(data);
worker.onmessage = (e) => updateUI(e.data);
```

### CLS Optimization Checklist

```css
/* ❌ Media without specified dimensions */
img { width: 100%; }

/* ✅ Reserve space */
img {
  width: 100%;
  aspect-ratio: 16 / 9;
}

/* ❌ Dynamically inserted content causing layout shift */
.ad-container { }

/* ✅ Reserve fixed height */
.ad-container {
  min-height: 250px;
}
```

**CLS review checklist:**
- [ ] Do images/videos have width/height or aspect-ratio?
- [ ] Does font loading use `font-display: swap`?
- [ ] Is space reserved for dynamic content?
- [ ] Are you avoiding inserting content above existing content?

---

## JavaScript Performance

### Code Splitting and Lazy Loading

```javascript
// ❌ Loading all code at once
import { HeavyChart } from './charts';
import { PDFExporter } from './pdf';
import { AdminPanel } from './admin';

// ✅ Load on demand
const HeavyChart = lazy(() => import('./charts'));
const PDFExporter = lazy(() => import('./pdf'));

// ✅ Route-level code splitting
const routes = [
  {
    path: '/dashboard',
    component: lazy(() => import('./pages/Dashboard')),
  },
  {
    path: '/admin',
    component: lazy(() => import('./pages/Admin')),
  },
];
```

### Bundle Size Optimization

```javascript
// ❌ Importing entire libraries
import _ from 'lodash';
import moment from 'moment';

// ✅ Import on demand
import debounce from 'lodash/debounce';
import { format } from 'date-fns';

// ❌ Not using Tree Shaking
export default {
  fn1() {},
  fn2() {},  // Unused but still bundled
};

// ✅ Named exports support Tree Shaking
export function fn1() {}
export function fn2() {}
```

**Bundle review checklist:**
- [ ] Is dynamic import() used for code splitting?
- [ ] Are large libraries imported on demand?
- [ ] Has bundle size been analyzed? (webpack-bundle-analyzer)
- [ ] Are there unused dependencies?

### List Rendering Optimization

```javascript
// ❌ Rendering a large list
function List({ items }) {
  return (
    <ul>
      {items.map(item => <li key={item.id}>{item.name}</li>)}
    </ul>
  );  // 10,000 items = 10,000 DOM nodes
}

// ✅ Virtual list - only renders visible items
import { FixedSizeList } from 'react-window';

function VirtualList({ items }) {
  return (
    <FixedSizeList
      height={400}
      itemCount={items.length}
      itemSize={35}
    >
      {({ index, style }) => (
        <div style={style}>{items[index].name}</div>
      )}
    </FixedSizeList>
  );
}
```

**Large data review points:**
- [ ] Are lists over 100 items using virtual scrolling?
- [ ] Does the table support pagination or virtualization?
- [ ] Are there unnecessary full renders?

---

## Memory Management

### Common Memory Leaks

#### 1. Uncleaned Event Listeners

```javascript
// ❌ Event still listening after component unmounts
useEffect(() => {
  window.addEventListener('resize', handleResize);
}, []);

// ✅ Clean up event listeners
useEffect(() => {
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, []);
```

#### 2. Uncleaned Timers

```javascript
// ❌ Timer not cleaned up
useEffect(() => {
  setInterval(fetchData, 5000);
}, []);

// ✅ Clean up timer
useEffect(() => {
  const timer = setInterval(fetchData, 5000);
  return () => clearInterval(timer);
}, []);
```

#### 3. Closure References

```javascript
// ❌ Closure holding reference to large object
function createHandler() {
  const largeData = new Array(1000000).fill('x');

  return function handler() {
    // largeData is referenced by closure, can't be garbage collected
    console.log(largeData.length);
  };
}

// ✅ Only keep necessary data
function createHandler() {
  const largeData = new Array(1000000).fill('x');
  const length = largeData.length;  // Only keep the needed value

  return function handler() {
    console.log(length);
  };
}
```

#### 4. Uncleaned Subscriptions

```javascript
// ❌ WebSocket/EventSource not closed
useEffect(() => {
  const ws = new WebSocket('wss://...');
  ws.onmessage = handleMessage;
}, []);

// ✅ Clean up connection
useEffect(() => {
  const ws = new WebSocket('wss://...');
  ws.onmessage = handleMessage;
  return () => ws.close();
}, []);
```

### Memory Review Checklist

```markdown
- [ ] Do all useEffects have cleanup functions?
- [ ] Are event listeners removed on component unmount?
- [ ] Are timers cleaned up?
- [ ] Are WebSocket/SSE connections closed?
- [ ] Are large objects released promptly?
- [ ] Are there global variables accumulating data?
```

### Detection Tools

| Tool | Purpose |
|------|---------|
| Chrome DevTools Memory | Heap snapshot analysis |
| MemLab (Meta) | Automated memory leak detection |
| Performance Monitor | Real-time memory monitoring |

---

## Database Performance

### N+1 Query Problem

```python
# ❌ N+1 problem - 1 + N queries
users = User.objects.all()  # 1 query
for user in users:
    print(user.profile.bio)  # N queries (one per user)

# ✅ Eager Loading - 2 queries
users = User.objects.select_related('profile').all()
for user in users:
    print(user.profile.bio)  # No additional queries

# ✅ Many-to-many with prefetch_related
posts = Post.objects.prefetch_related('tags').all()
```

```javascript
// TypeORM example
// ❌ N+1 problem
const users = await userRepository.find();
for (const user of users) {
  const posts = await user.posts;  // Query each iteration
}

// ✅ Eager Loading
const users = await userRepository.find({
  relations: ['posts'],
});
```

### Index Optimization

```sql
-- ❌ Full table scan
SELECT * FROM orders WHERE status = 'pending';

-- ✅ Add index
CREATE INDEX idx_orders_status ON orders(status);

-- ❌ Index invalidation: Function on column
SELECT * FROM users WHERE YEAR(created_at) = 2024;

-- ✅ Range query uses index
SELECT * FROM users
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- ❌ Index invalidation: LIKE with leading wildcard
SELECT * FROM products WHERE name LIKE '%phone%';

-- ✅ Prefix match uses index
SELECT * FROM products WHERE name LIKE 'phone%';
```

### Query Optimization

```sql
-- ❌ SELECT * fetching unnecessary columns
SELECT * FROM users WHERE id = 1;

-- ✅ Only fetch needed columns
SELECT id, name, email FROM users WHERE id = 1;

-- ❌ Large table without LIMIT
SELECT * FROM logs WHERE type = 'error';

-- ✅ Paginated query
SELECT * FROM logs WHERE type = 'error' LIMIT 100 OFFSET 0;

-- ❌ Query in a loop
for id in user_ids:
    cursor.execute("SELECT * FROM users WHERE id = %s", (id,))

-- ✅ Batch query
cursor.execute("SELECT * FROM users WHERE id IN %s", (tuple(user_ids),))
```

### Database Review Checklist

```markdown
🔴 Must check:
- [ ] Are there N+1 queries?
- [ ] Are WHERE clause columns indexed?
- [ ] Is SELECT * avoided?
- [ ] Do large table queries have LIMIT?

🟡 Should check:
- [ ] Has EXPLAIN been used to analyze query plans?
- [ ] Is the composite index column order correct?
- [ ] Are there unused indexes?
- [ ] Is there slow query log monitoring?
```

---

## API Performance

### Pagination Implementation

```javascript
// ❌ Returning all data
app.get('/users', async (req, res) => {
  const users = await User.findAll();  // Could return 100,000 records
  res.json(users);
});

// ✅ Pagination + max limit
app.get('/users', async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = Math.min(parseInt(req.query.limit) || 20, 100);  // Max 100
  const offset = (page - 1) * limit;

  const { rows, count } = await User.findAndCountAll({
    limit,
    offset,
    order: [['id', 'ASC']],
  });

  res.json({
    data: rows,
    pagination: {
      page,
      limit,
      total: count,
      totalPages: Math.ceil(count / limit),
    },
  });
});
```

### Caching Strategy

```javascript
// ✅ Redis cache example
async function getUser(id) {
  const cacheKey = `user:${id}`;

  // 1. Check cache
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  // 2. Query database
  const user = await db.users.findById(id);

  // 3. Write to cache (with expiration)
  await redis.setex(cacheKey, 3600, JSON.stringify(user));

  return user;
}

// ✅ HTTP cache headers
app.get('/static-data', (req, res) => {
  res.set({
    'Cache-Control': 'public, max-age=86400',  // 24 hours
    'ETag': 'abc123',
  });
  res.json(data);
});
```

### Response Compression

```javascript
// ✅ Enable Gzip/Brotli compression
const compression = require('compression');
app.use(compression());

// ✅ Return only necessary fields
// Request: GET /users?fields=id,name,email
app.get('/users', async (req, res) => {
  const fields = req.query.fields?.split(',') || ['id', 'name'];
  const users = await User.findAll({
    attributes: fields,
  });
  res.json(users);
});
```

### Rate Limiting

```javascript
// ✅ Rate limiting
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 60 * 1000,  // 1 minute
  max: 100,             // Max 100 requests
  message: { error: 'Too many requests, please try again later.' },
});

app.use('/api/', limiter);
```

### API Review Checklist

```markdown
- [ ] Do list endpoints have pagination?
- [ ] Is there a max per-page limit?
- [ ] Is hot data cached?
- [ ] Is response compression enabled?
- [ ] Is there rate limiting?
- [ ] Are only necessary fields returned?
```

---

## Algorithm Complexity

### Common Complexity Comparison

| Complexity | Name | 10 items | 1000 items | 1M items | Example |
|------------|------|----------|------------|----------|---------|
| O(1) | Constant | 1 | 1 | 1 | Hash lookup |
| O(log n) | Logarithmic | 3 | 10 | 20 | Binary search |
| O(n) | Linear | 10 | 1000 | 1M | Array traversal |
| O(n log n) | Linearithmic | 33 | 10,000 | 20M | Quick sort |
| O(n²) | Quadratic | 100 | 1M | 1T | Nested loops |
| O(2ⁿ) | Exponential | 1024 | ∞ | ∞ | Recursive Fibonacci |

### Recognition in Code Review

```javascript
// ❌ O(n²) - Nested loop
function findDuplicates(arr) {
  const duplicates = [];
  for (let i = 0; i < arr.length; i++) {
    for (let j = i + 1; j < arr.length; j++) {
      if (arr[i] === arr[j]) {
        duplicates.push(arr[i]);
      }
    }
  }
  return duplicates;
}

// ✅ O(n) - Using Set
function findDuplicates(arr) {
  const seen = new Set();
  const duplicates = new Set();
  for (const item of arr) {
    if (seen.has(item)) {
      duplicates.add(item);
    }
    seen.add(item);
  }
  return [...duplicates];
}
```

```javascript
// ❌ O(n²) - Using includes in every iteration
function removeDuplicates(arr) {
  const result = [];
  for (const item of arr) {
    if (!result.includes(item)) {  // includes is O(n)
      result.push(item);
    }
  }
  return result;
}

// ✅ O(n) - Using Set
function removeDuplicates(arr) {
  return [...new Set(arr)];
}
```

```javascript
// ❌ O(n) lookup - Traversing every time
const users = [{ id: 1, name: 'A' }, { id: 2, name: 'B' }, ...];

function getUser(id) {
  return users.find(u => u.id === id);  // O(n)
}

// ✅ O(1) lookup - Using Map
const userMap = new Map(users.map(u => [u.id, u]));

function getUser(id) {
  return userMap.get(id);  // O(1)
}
```

### Space Complexity Considerations

```javascript
// ⚠️ O(n) space - Creating new array
const doubled = arr.map(x => x * 2);

// ✅ O(1) space - In-place modification (if allowed)
for (let i = 0; i < arr.length; i++) {
  arr[i] *= 2;
}

// ⚠️ Deep recursion may cause stack overflow
function factorial(n) {
  if (n <= 1) return 1;
  return n * factorial(n - 1);  // O(n) stack space
}

// ✅ Iterative version O(1) space
function factorial(n) {
  let result = 1;
  for (let i = 2; i <= n; i++) {
    result *= i;
  }
  return result;
}
```

### Complexity Review Examples:

```markdown
💡 "This nested loop has O(n²) complexity, which will cause performance issues with large datasets"
🔴 "Using Array.includes() in a loop makes this O(n²), recommend using Set instead"
🟡 "This recursion depth may cause stack overflow, consider iterative or tail-recursive version"
```

---

## Performance Review Checklist

### 🔴 Must Check (Blocking)

**Frontend:**
- [ ] Is the LCP image lazy-loaded? (It shouldn't be)
- [ ] Is there `transition: all`?
- [ ] Are width/height/top/left being animated?
- [ ] Are lists with >100 items virtualized?

**Backend:**
- [ ] Are there N+1 queries?
- [ ] Do list endpoints have pagination?
- [ ] Is SELECT * used on large tables?

**General:**
- [ ] Are there O(n²) or worse nested loops?
- [ ] Do useEffects/event listeners have cleanup?

### 🟡 Should Check (Important)

**Frontend:**
- [ ] Is code splitting used?
- [ ] Are large libraries imported on demand?
- [ ] Are images using WebP/AVIF?
- [ ] Are there unused dependencies?

**Backend:**
- [ ] Is hot data cached?
- [ ] Are WHERE columns indexed?
- [ ] Is there slow query monitoring?

**API:**
- [ ] Is response compression enabled?
- [ ] Is there rate limiting?
- [ ] Are only necessary fields returned?

### 🟢 Optimization Suggestions (Nice to have)

- [ ] Has bundle size been analyzed?
- [ ] Is CDN used?
- [ ] Is there performance monitoring?
- [ ] Have performance benchmarks been run?

---

## Performance Thresholds

### Frontend Metrics

| Metric | Good | Needs Improvement | Poor |
|--------|------|-------------------|------|
| LCP | ≤ 2.5s | 2.5-4s | > 4s |
| INP | ≤ 200ms | 200-500ms | > 500ms |
| CLS | ≤ 0.1 | 0.1-0.25 | > 0.25 |
| FCP | ≤ 1.8s | 1.8-3s | > 3s |
| Bundle Size (JS) | < 200KB | 200-500KB | > 500KB |

### Backend Metrics

| Metric | Good | Needs Improvement | Poor |
|--------|------|-------------------|------|
| API Response Time | < 100ms | 100-500ms | > 500ms |
| Database Query | < 50ms | 50-200ms | > 200ms |
| Page Load | < 3s | 3-5s | > 5s |

---

## Tool Recommendations

### Frontend Performance

| Tool | Purpose |
|------|---------|
| [Lighthouse](https://developer.chrome.com/docs/lighthouse/) | Core Web Vitals testing |
| [WebPageTest](https://www.webpagetest.org/) | Detailed performance analysis |
| [webpack-bundle-analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer) | Bundle analysis |
| [Chrome DevTools Performance](https://developer.chrome.com/docs/devtools/performance/) | Runtime performance analysis |

### Memory Detection

| Tool | Purpose |
|------|---------|
| [MemLab](https://github.com/facebookincubator/memlab) | Automated memory leak detection |
| Chrome Memory Tab | Heap snapshot analysis |

### Backend Performance

| Tool | Purpose |
|------|---------|
| EXPLAIN | Database query plan analysis |
| [pganalyze](https://pganalyze.com/) | PostgreSQL performance monitoring |
| [New Relic](https://newrelic.com/) / [Datadog](https://www.datadoghq.com/) | APM monitoring |

---

## Reference Resources

- [Core Web Vitals - web.dev](https://web.dev/articles/vitals)
- [Optimizing Core Web Vitals - Vercel](https://vercel.com/guides/optimizing-core-web-vitals-in-2024)
- [MemLab - Meta Engineering](https://engineering.fb.com/2022/09/12/open-source/memlab/)
- [Big O Cheat Sheet](https://www.bigocheatsheet.com/)
- [N+1 Query Problem - Stack Overflow](https://stackoverflow.com/questions/97197/what-is-the-n1-selects-problem-in-orm-object-relational-mapping)
- [API Performance Optimization](https://algorithmsin60days.com/blog/optimizing-api-performance/)
