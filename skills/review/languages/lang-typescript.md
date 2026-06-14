# TypeScript/JavaScript Code Review Guide

> TypeScript code review guide covering the type system, generics, conditional types, strict mode, async/await patterns, and more.

## Table of Contents

- [Type Safety Basics](#type-safety-basics)
- [Generic Patterns](#generic-patterns)
- [Advanced Types](#advanced-types)
- [Strict Mode Configuration](#strict-mode-configuration)
- [Async Handling](#async-handling)
- [Immutability](#immutability)
- [ESLint Rules](#eslint-rules)
- [Review Checklist](#review-checklist)

---

## Type Safety Basics

### Avoid Using any

```typescript
// ❌ Using any defeats type safety
function processData(data: any) {
  return data.value;  // No type checking, may crash at runtime
}

// ✅ Use proper types
interface DataPayload {
  value: string;
}
function processData(data: DataPayload) {
  return data.value;
}

// ✅ For unknown types, use unknown + type guards
function processUnknown(data: unknown) {
  if (typeof data === 'object' && data !== null && 'value' in data) {
    return (data as { value: string }).value;
  }
  throw new Error('Invalid data');
}
```

### Type Narrowing

```typescript
// ❌ Unsafe type assertion
function getLength(value: string | string[]) {
  return (value as string[]).length;  // Will error if value is a string
}

// ✅ Use type guards
function getLength(value: string | string[]): number {
  if (Array.isArray(value)) {
    return value.length;
  }
  return value.length;
}

// ✅ Use in operator
interface Dog { bark(): void }
interface Cat { meow(): void }

function speak(animal: Dog | Cat) {
  if ('bark' in animal) {
    animal.bark();
  } else {
    animal.meow();
  }
}
```

### Literal Types and as const

```typescript
// ❌ Type too broad
const config = {
  endpoint: '/api',
  method: 'GET'  // Type is string
};

// ✅ Use as const for literal types
const config = {
  endpoint: '/api',
  method: 'GET'
} as const;  // method type is 'GET'

// ✅ For function parameters
function request(method: 'GET' | 'POST', url: string) { ... }
request(config.method, config.endpoint);  // Correct!
```

---

## Generic Patterns

### Basic Generics

```typescript
// ❌ Duplicate code
function getFirstString(arr: string[]): string | undefined {
  return arr[0];
}
function getFirstNumber(arr: number[]): number | undefined {
  return arr[0];
}

// ✅ Use generics
function getFirst<T>(arr: T[]): T | undefined {
  return arr[0];
}
```

### Generic Constraints

```typescript
// ❌ Generic without constraint, can't access properties
function getProperty<T>(obj: T, key: string) {
  return obj[key];  // Error: Can't index
}

// ✅ Use keyof for constraint
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: 'Alice', age: 30 };
getProperty(user, 'name');  // Return type is string
getProperty(user, 'age');   // Return type is number
getProperty(user, 'foo');   // Error: 'foo' not in keyof User
```

### Generic Default Values

```typescript
// ✅ Provide reasonable default types
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
  message: string;
}

// Can omit generic parameter
const response: ApiResponse = { data: null, status: 200, message: 'OK' };
// Or specify
const userResponse: ApiResponse<User> = { ... };
```

### Common Generic Utility Types

```typescript
// ✅ Make good use of built-in utility types
interface User {
  id: number;
  name: string;
  email: string;
}

type PartialUser = Partial<User>;         // All properties optional
type RequiredUser = Required<User>;       // All properties required
type ReadonlyUser = Readonly<User>;       // All properties readonly
type UserKeys = keyof User;               // 'id' | 'name' | 'email'
type NameOnly = Pick<User, 'name'>;       // { name: string }
type WithoutId = Omit<User, 'id'>;        // { name: string; email: string }
type UserRecord = Record<string, User>;   // { [key: string]: User }
```

---

## Advanced Types

### Conditional Types

```typescript
// ✅ Return different types based on input type
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;  // true
type B = IsString<number>;  // false

// ✅ Extract array element type
type ElementType<T> = T extends (infer U)[] ? U : never;

type Elem = ElementType<string[]>;  // string

// ✅ Extract function return type (built-in ReturnType)
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
```

### Mapped Types

```typescript
// ✅ Transform all properties of an object type
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};

interface User {
  name: string;
  age: number;
}

type NullableUser = Nullable<User>;
// { name: string | null; age: number | null }

// ✅ Add prefix
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number }
```

### Template Literal Types

```typescript
// ✅ Type-safe event names
type EventName = 'click' | 'focus' | 'blur';
type HandlerName = `on${Capitalize<EventName>}`;
// 'onClick' | 'onFocus' | 'onBlur'

// ✅ API route types
type ApiRoute = `/api/${string}`;
const route: ApiRoute = '/api/users';  // OK
const badRoute: ApiRoute = '/users';   // Error
```

### Discriminated Unions

```typescript
// ✅ Use discriminant property for type safety
type Result<T, E> =
  | { success: true; data: T }
  | { success: false; error: E };

function handleResult(result: Result<User, Error>) {
  if (result.success) {
    console.log(result.data.name);  // TypeScript knows data exists
  } else {
    console.log(result.error.message);  // TypeScript knows error exists
  }
}

// ✅ Redux Action pattern
type Action =
  | { type: 'INCREMENT'; payload: number }
  | { type: 'DECREMENT'; payload: number }
  | { type: 'RESET' };

function reducer(state: number, action: Action): number {
  switch (action.type) {
    case 'INCREMENT':
      return state + action.payload;  // payload type known
    case 'DECREMENT':
      return state - action.payload;
    case 'RESET':
      return 0;  // No payload here
  }
}
```

---

## Strict Mode Configuration

### Recommended tsconfig.json (TypeScript 5.x+)

```json
{
  "compilerOptions": {
    // ✅ Required strict options
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "useUnknownInCatchVariables": true,

    // ✅ Additional recommended options
    "noUncheckedIndexedAccess": true,     // Treat arr[i] as T | undefined
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,   // Don't allow undefined in optional props
    "noPropertyAccessFromIndexSignature": true,
    "moduleResolution": "bundler",        // For Vite/Next.js 13+ projects
    "verbatimModuleSyntax": true          // Prevents import type hoisting issues
  }
}
```

### Key Strict Options Explained

| Option | What it does | Why it matters |
|--------|-------------|----------------|
| `noUncheckedIndexedAccess` | `arr[i]` is `T \| undefined` | Prevents runtime crashes from out-of-bounds access |
| `exactOptionalPropertyTypes` | Optional props can't be set to `undefined` | Distinguishes "not set" from "set to undefined" |
| `useUnknownInCatchVariables` | `catch (e)` is `unknown` not `any` | Forces proper error type checking |
| `verbatimModuleSyntax` | Preserves import/export kinds | Prevents runtime errors from hoisted imports |

### noUncheckedIndexedAccess Impact

```typescript
// tsconfig: "noUncheckedIndexedAccess": true

const arr = [1, 2, 3];
const first = arr[0];  // Type is number | undefined

// ❌ Using directly may error
console.log(first.toFixed(2));  // Error: May be undefined

// ✅ Check first
if (first !== undefined) {
  console.log(first.toFixed(2));
}

// ✅ Or use non-null assertion when certain
console.log(arr[0]!.toFixed(2));
```

---

## Async Handling

### Promise Error Handling

```typescript
// ❌ Not handling async errors
async function fetchUser(id: string) {
  const response = await fetch(`/api/users/${id}`);
  return response.json();  // Network errors not handled
}

// ✅ Handle errors properly
async function fetchUser(id: string): Promise<User> {
  try {
    const response = await fetch(`/api/users/${id}`);
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    return await response.json();
  } catch (error) {
    if (error instanceof Error) {
      throw new Error(`Failed to fetch user: ${error.message}`);
    }
    throw error;
  }
}
```

### Promise.all vs Promise.allSettled

```typescript
// ❌ Promise.all fails entirely on first failure
async function fetchAllUsers(ids: string[]) {
  const users = await Promise.all(ids.map(fetchUser));
  return users;  // One failure and all fail
}

// ✅ Promise.allSettled to get all results
async function fetchAllUsers(ids: string[]) {
  const results = await Promise.allSettled(ids.map(fetchUser));

  const users: User[] = [];
  const errors: Error[] = [];

  for (const result of results) {
    if (result.status === 'fulfilled') {
      users.push(result.value);
    } else {
      errors.push(result.reason);
    }
  }

  return { users, errors };
}
```

### Race Condition Handling

```typescript
// ❌ Race condition: Old request may overwrite new request
function useSearch() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  useEffect(() => {
    fetch(`/api/search?q=${query}`)
      .then(r => r.json())
      .then(setResults);  // Old request may return later!
  }, [query]);
}

// ✅ Use AbortController
function useSearch() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  useEffect(() => {
    const controller = new AbortController();

    fetch(`/api/search?q=${query}`, { signal: controller.signal })
      .then(r => r.json())
      .then(setResults)
      .catch(e => {
        if (e.name !== 'AbortError') throw e;
      });

    return () => controller.abort();
  }, [query]);
}
```

---

## Immutability

### Readonly and ReadonlyArray

```typescript
// ❌ Mutable parameter may be accidentally modified
function processUsers(users: User[]) {
  users.sort((a, b) => a.name.localeCompare(b.name));  // Modified original array!
  return users;
}

// ✅ Use readonly to prevent modification
function processUsers(users: readonly User[]): User[] {
  return [...users].sort((a, b) => a.name.localeCompare(b.name));
}

// ✅ Deep readonly
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};
```

### Immutable Function Parameters

```typescript
// ✅ Use as const and readonly to protect data
function createConfig<T extends readonly string[]>(routes: T) {
  return routes;
}

const routes = createConfig(['home', 'about', 'contact'] as const);
// Type is readonly ['home', 'about', 'contact']
```

---

## ESLint Rules

### Recommended @typescript-eslint Rules

```javascript
// .eslintrc.js
module.exports = {
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
    'plugin:@typescript-eslint/recommended-requiring-type-checking',
    'plugin:@typescript-eslint/strict'
  ],
  rules: {
    // ✅ Type safety
    '@typescript-eslint/no-explicit-any': 'error',
    '@typescript-eslint/no-unsafe-assignment': 'error',
    '@typescript-eslint/no-unsafe-member-access': 'error',
    '@typescript-eslint/no-unsafe-call': 'error',
    '@typescript-eslint/no-unsafe-return': 'error',

    // ✅ Best practices
    '@typescript-eslint/explicit-function-return-type': 'warn',
    '@typescript-eslint/no-floating-promises': 'error',
    '@typescript-eslint/await-thenable': 'error',
    '@typescript-eslint/no-misused-promises': 'error',

    // ✅ Code style
    '@typescript-eslint/consistent-type-imports': 'error',
    '@typescript-eslint/prefer-nullish-coalescing': 'error',
    '@typescript-eslint/prefer-optional-chain': 'error'
  }
};
```

### Common ESLint Error Fixes

```typescript
// ❌ no-floating-promises: Promise must be handled
async function save() { ... }
save();  // Error: Unhandled Promise

// ✅ Handle explicitly
await save();
// Or
save().catch(console.error);
// Or explicitly ignore
void save();

// ❌ no-misused-promises: Can't use Promise in non-async position
const items = [1, 2, 3];
items.forEach(async (item) => {  // Error!
  await processItem(item);
});

// ✅ Use for...of
for (const item of items) {
  await processItem(item);
}
// Or Promise.all
await Promise.all(items.map(processItem));
```

---

## Review Checklist

### Type System
- [ ] No `any` used (use `unknown` + type guards instead)
- [ ] Interfaces and type definitions are complete and meaningfully named
- [ ] Generics used for code reusability
- [ ] Union types have correct type narrowing
- [ ] Make good use of utility types (Partial, Pick, Omit, etc.)

### Generics
- [ ] Generics have appropriate constraints (extends)
- [ ] Generic parameters have reasonable defaults
- [ ] Avoid over-generification (KISS principle)

### Strict Mode
- [ ] tsconfig.json has strict: true enabled
- [ ] noUncheckedIndexedAccess enabled
- [ ] No @ts-ignore used (use @ts-expect-error instead)

### Async Code
- [ ] async functions have error handling
- [ ] Promise rejections properly handled
- [ ] No floating promises (unhandled Promises)
- [ ] Concurrent requests use Promise.all or Promise.allSettled
- [ ] Race conditions handled with AbortController

### Immutability
- [ ] Don't directly modify function parameters
- [ ] Use spread operator to create new objects/arrays
- [ ] Consider using readonly modifier

### ESLint
- [ ] Using @typescript-eslint/recommended
- [ ] No ESLint warnings or errors
- [ ] Using consistent-type-imports
