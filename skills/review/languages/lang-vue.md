# Vue 3 Code Review Guide

> Vue 3 Composition API code review guide, covering the reactivity system, Props/Emits, Watchers, Composables, and Vue 3.5+ features (all stable).

## Table of Contents

- [Reactivity System](#reactivity-system)
- [Props & Emits](#props--emits)
- [Vue 3.5 New Features](#vue-35-new-features)
- [Watchers](#watchers)
- [Template Best Practices](#template-best-practices)
- [Composables](#composables)
- [Performance Optimization](#performance-optimization)
- [Review Checklist](#review-checklist)

---

## Reactivity System

### ref vs reactive Selection

```vue
<!-- ✅ Use ref for primitive types -->
<script setup lang="ts">
const count = ref(0)
const name = ref('Vue')

// ref needs .value access
count.value++
</script>

<!-- ✅ Use reactive for objects/arrays (optional)-->
<script setup lang="ts">
const state = reactive({
  user: null,
  loading: false,
  error: null
})

// reactive accessed directly
state.loading = true
</script>

<!-- 💡 Modern best practice: Use ref consistently for uniformity -->
<script setup lang="ts">
const user = ref<User | null>(null)
const loading = ref(false)
const error = ref<Error | null>(null)
</script>
```

### Destructuring reactive Objects

```vue
<!-- ❌ Destructuring reactive loses reactivity -->
<script setup lang="ts">
const state = reactive({ count: 0, name: 'Vue' })
const { count, name } = state  // Loses reactivity!
</script>

<!-- ✅ Use toRefs to maintain reactivity -->
<script setup lang="ts">
const state = reactive({ count: 0, name: 'Vue' })
const { count, name } = toRefs(state)  // Maintains reactivity
// Or just use refs directly
const count = ref(0)
const name = ref('Vue')
</script>
```

### computed Side Effects

```vue
<!-- ❌ Side effects in computed -->
<script setup lang="ts">
const fullName = computed(() => {
  console.log('Computing...')  // Side effect!
  otherRef.value = 'changed'   // Modifying other state!
  return `${firstName.value} ${lastName.value}`
})
</script>

<!-- ✅ computed only for derived state -->
<script setup lang="ts">
const fullName = computed(() => {
  return `${firstName.value} ${lastName.value}`
})
// Side effects in watch or event handlers
watch(fullName, (name) => {
  console.log('Name changed:', name)
})
</script>
```

### shallowRef Optimization

```vue
<!-- ❌ Using ref for large objects does deep conversion -->
<script setup lang="ts">
const largeData = ref(hugeNestedObject)  // Deep reactivity, high performance cost
</script>

<!-- ✅ Use shallowRef to avoid deep conversion -->
<script setup lang="ts">
const largeData = shallowRef(hugeNestedObject)

// Only triggers update when replacing the whole object
function updateData(newData) {
  largeData.value = newData  // ✅ Triggers update
}

// ❌ Modifying nested properties won't trigger updates
// largeData.value.nested.prop = 'new'

// Use triggerRef when manual triggering is needed
import { triggerRef } from 'vue'
largeData.value.nested.prop = 'new'
triggerRef(largeData)
</script>
```

---

## Props & Emits

### Directly Modifying Props

```vue
<!-- ❌ Directly modifying props -->
<script setup lang="ts">
const props = defineProps<{ user: User }>()
props.user.name = 'New Name'  // Never directly modify props!
</script>

<!-- ✅ Use emit to notify parent component to update -->
<script setup lang="ts">
const props = defineProps<{ user: User }>()
const emit = defineEmits<{
  update: [name: string]
}>()
const updateName = (name: string) => emit('update', name)
</script>
```

### defineProps Type Declarations

```vue
<!-- ❌ defineProps missing type declarations -->
<script setup lang="ts">
const props = defineProps(['title', 'count'])  // No type checking
</script>

<!-- ✅ Use type declarations + withDefaults -->
<script setup lang="ts">
interface Props {
  title: string
  count?: number
  items?: string[]
}
const props = withDefaults(defineProps<Props>(), {
  count: 0,
  items: () => []  // Object/array defaults need factory functions
})
</script>
```

### defineEmits Type Safety

```vue
<!-- ❌ defineEmits missing types -->
<script setup lang="ts">
const emit = defineEmits(['update', 'delete'])  // No type checking
emit('update', someValue)  // Parameter types not safe
</script>

<!-- ✅ Complete type definitions -->
<script setup lang="ts">
const emit = defineEmits<{
  update: [id: number, value: string]
  delete: [id: number]
  'custom-event': [payload: CustomPayload]
}>()

// Now have full type checking
emit('update', 1, 'new value')  // ✅
emit('update', 'wrong')  // ❌ TypeScript error
</script>
```

---

## Vue 3.5 New Features

### Reactive Props Destructure (3.5+)

```vue
<!-- Before Vue 3.5: Destructuring loses reactivity -->
<script setup lang="ts">
const props = defineProps<{ count: number }>()
// Need to use props.count or toRefs
</script>

<!-- ✅ Vue 3.5+: Destructuring maintains reactivity -->
<script setup lang="ts">
const { count, name = 'default' } = defineProps<{
  count: number
  name?: string
}>()

// count and name automatically maintain reactivity!
// Can use directly in templates and watch
watch(() => count, (newCount) => {
  console.log('Count changed:', newCount)
})
</script>

<!-- ✅ With defaults -->
<script setup lang="ts">
const {
  title,
  count = 0,
  items = () => []  // Function as default (for objects/arrays)
} = defineProps<{
  title: string
  count?: number
  items?: () => string[]
}>()
</script>
```

### defineModel (3.4+)

```vue
<!-- ❌ Traditional v-model implementation: Verbose -->
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()

// Need computed for two-way binding
const value = computed({
  get: () => props.modelValue,
  set: (val) => emit('update:modelValue', val)
})
</script>

<!-- ✅ defineModel: Concise v-model implementation -->
<script setup lang="ts">
// Automatically handles props and emit
const model = defineModel<string>()

// Use directly
model.value = 'new value'  // Auto emits
</script>
<template>
  <input v-model="model" />
</template>

<!-- ✅ Named v-model -->
<script setup lang="ts">
// Implementation of v-model:title
const title = defineModel<string>('title')

// With defaults and options
const count = defineModel<number>('count', {
  default: 0,
  required: false
})
</script>

<!-- ✅ Multiple v-model -->
<script setup lang="ts">
const firstName = defineModel<string>('firstName')
const lastName = defineModel<string>('lastName')
</script>
<template>
  <!-- Parent usage: <MyInput v-model:first-name="first" v-model:last-name="last" /> -->
</template>

<!-- ✅ v-model modifiers -->
<script setup lang="ts">
const [model, modifiers] = defineModel<string>()

// Check modifiers
if (modifiers.capitalize) {
  // Handle .capitalize modifier
}
</script>
```

### useTemplateRef (3.5+)

```vue
<!-- Traditional approach: ref attribute matches variable name -->
<script setup lang="ts">
const inputRef = ref<HTMLInputElement | null>(null)
</script>
<template>
  <input ref="inputRef" />
</template>

<!-- ✅ useTemplateRef: Clearer template references -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'

const input = useTemplateRef<HTMLInputElement>('my-input')

onMounted(() => {
  input.value?.focus()
})
</script>
<template>
  <input ref="my-input" />
</template>

<!-- ✅ Dynamic ref -->
<script setup lang="ts">
const refKey = ref('input-a')
const dynamicInput = useTemplateRef<HTMLInputElement>(refKey)
</script>
```

### useId (3.5+)

```vue
<!-- ❌ Manually generated IDs may conflict -->
<script setup lang="ts">
const id = `input-${Math.random()}`  // SSR inconsistent!
</script>

<!-- ✅ useId: SSR-safe unique IDs -->
<script setup lang="ts">
import { useId } from 'vue'

const id = useId()  // e.g., 'v-0'
</script>
<template>
  <label :for="id">Name</label>
  <input :id="id" />
</template>

<!-- ✅ Usage in form components -->
<script setup lang="ts">
const inputId = useId()
const errorId = useId()
</script>
<template>
  <label :for="inputId">Email</label>
  <input
    :id="inputId"
    :aria-describedby="errorId"
  />
  <span :id="errorId" class="error">{{ error }}</span>
</template>
```

### onWatcherCleanup (3.5+)

```vue
<!-- Traditional approach: watch third parameter -->
<script setup lang="ts">
watch(source, async (value, oldValue, onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort())
  // ...
})
</script>

<!-- ✅ onWatcherCleanup: More flexible cleanup -->
<script setup lang="ts">
import { onWatcherCleanup } from 'vue'

watch(source, async (value) => {
  const controller = new AbortController()
  onWatcherCleanup(() => controller.abort())

  // Can be called anywhere, not limited to callback start
  if (someCondition) {
    const anotherResource = createResource()
    onWatcherCleanup(() => anotherResource.dispose())
  }

  await fetchData(value, controller.signal)
})
</script>
```

### Deferred Teleport (3.5+)

```vue
<!-- ❌ Teleport target must exist at mount time -->
<template>
  <Teleport to="#modal-container">
    <!-- Will error if #modal-container doesn't exist -->
  </Teleport>
</template>

<!-- ✅ defer attribute delays mounting -->
<template>
  <Teleport to="#modal-container" defer>
    <!-- Wait for target element to exist before mounting -->
    <Modal />
  </Teleport>
</template>
```

---

## Watchers

### watch vs watchEffect

```vue
<script setup lang="ts">
// ✅ watch: Explicitly specify dependencies, lazy execution
watch(
  () => props.userId,
  async (userId) => {
    user.value = await fetchUser(userId)
  }
)

// ✅ watchEffect: Auto-collect dependencies, execute immediately
watchEffect(async () => {
  // Auto-tracks props.userId
  user.value = await fetchUser(props.userId)
})

// 💡 Selection guide:
// - Need old value? Use watch
// - Need lazy execution? Use watch
// - Dependencies complex? Use watchEffect
</script>
```

### watch Cleanup Functions

```vue
<!-- ❌ watch missing cleanup, may memory leak -->
<script setup lang="ts">
watch(searchQuery, async (query) => {
  const controller = new AbortController()
  const data = await fetch(`/api/search?q=${query}`, {
    signal: controller.signal
  })
  results.value = await data.json()
  // If query changes quickly, old requests won't be cancelled!
})
</script>

<!-- ✅ Use onCleanup to clean up side effects -->
<script setup lang="ts">
watch(searchQuery, async (query, _, onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort())  // Cancel old request

  try {
    const data = await fetch(`/api/search?q=${query}`, {
      signal: controller.signal
    })
    results.value = await data.json()
  } catch (e) {
    if (e.name !== 'AbortError') throw e
  }
})
</script>
```

### watch Options

```vue
<script setup lang="ts">
// ✅ immediate: Execute once immediately
watch(
  userId,
  async (id) => {
    user.value = await fetchUser(id)
  },
  { immediate: true }
)

// ✅ deep: Deep listening (performance cost, use cautiously)
watch(
  state,
  (newState) => {
    console.log('State changed deeply')
  },
  { deep: true }
)

// ✅ flush: 'post': Execute after DOM updates
watch(
  source,
  () => {
    // Can safely access updated DOM
    // nextTick no longer needed
  },
  { flush: 'post' }
)

// ✅ once: true (Vue 3.4+): Execute only once
watch(
  source,
  (value) => {
    console.log('Will execute only once:', value)
  },
  { once: true }
)
</script>
```

### Listening to Multiple Sources

```vue
<script setup lang="ts">
// ✅ Listen to multiple refs
watch(
  [firstName, lastName],
  ([newFirst, newLast], [oldFirst, oldLast]) => {
    console.log(`Name changed from ${oldFirst} ${oldLast} to ${newFirst} ${newLast}`)
  }
)

// ✅ Listen to specific properties of reactive object
watch(
  () => [state.count, state.name],
  ([count, name]) => {
    console.log(`count: ${count}, name: ${name}`)
  }
)
</script>
```

---

## Template Best Practices

### v-for Keys

```vue
<!-- ❌ Using index as key in v-for -->
<template>
  <li v-for="(item, index) in items" :key="index">
    {{ item.name }}
  </li>
</template>

<!-- ✅ Use unique identifier as key -->
<template>
  <li v-for="item in items" :key="item.id">
    {{ item.name }}
  </li>
</template>

<!-- ✅ Compound key (when no unique ID available)-->
<template>
  <li v-for="(item, index) in items" :key="`${item.name}-${item.type}-${index}`">
    {{ item.name }}
  </li>
</template>
```

### v-if and v-for Priority

```vue
<!-- ❌ v-if and v-for used together -->
<template>
  <li v-for="user in users" v-if="user.active" :key="user.id">
    {{ user.name }}
  </li>
</template>

<!-- ✅ Use computed to filter -->
<script setup lang="ts">
const activeUsers = computed(() =>
  users.value.filter(user => user.active)
)
</script>
<template>
  <li v-for="user in activeUsers" :key="user.id">
    {{ user.name }}
  </li>
</template>

<!-- ✅ Or wrap with template -->
<template>
  <template v-for="user in users" :key="user.id">
    <li v-if="user.active">
      {{ user.name }}
    </li>
  </template>
</template>
```

### Event Handling

```vue
<!-- ❌ Inline complex logic -->
<template>
  <button @click="items = items.filter(i => i.id !== item.id); count--">
    Delete
  </button>
</template>

<!-- ✅ Use methods -->
<script setup lang="ts">
const deleteItem = (id: number) => {
  items.value = items.value.filter(i => i.id !== id)
  count.value--
}
</script>
<template>
  <button @click="deleteItem(item.id)">Delete</button>
</template>

<!-- ✅ Event modifiers -->
<template>
  <!-- Prevent default behavior -->
  <form @submit.prevent="handleSubmit">...</form>

  <!-- Stop propagation -->
  <button @click.stop="handleClick">...</button>

  <!-- Execute only once -->
  <button @click.once="handleOnce">...</button>

  <!-- Keyboard modifiers -->
  <input @keyup.enter="submit" @keyup.esc="cancel" />
</template>
```

---

## Composables

### Composable Design Principles

```typescript
// ✅ Good composable design
export function useCounter(initialValue = 0) {
  const count = ref(initialValue)

  const increment = () => count.value++
  const decrement = () => count.value--
  const reset = () => count.value = initialValue

  // Return reactive refs and methods
  return {
    count: readonly(count),  // Read-only prevents external modification
    increment,
    decrement,
    reset
  }
}

// ❌ Don't return .value
export function useBadCounter() {
  const count = ref(0)
  return {
    count: count.value  // ❌ Loses reactivity!
  }
}
```

### Passing Props to Composables

```vue
<!-- ❌ Passing props to composable loses reactivity -->
<script setup lang="ts">
const props = defineProps<{ userId: string }>()
const { user } = useUser(props.userId)  // Loses reactivity!
</script>

<!-- ✅ Use toRef or computed to maintain reactivity -->
<script setup lang="ts">
const props = defineProps<{ userId: string }>()
const userIdRef = toRef(props, 'userId')
const { user } = useUser(userIdRef)  // Maintains reactivity
// Or use computed
const { user } = useUser(computed(() => props.userId))

// ✅ Vue 3.5+: Direct destructuring usage
const { userId } = defineProps<{ userId: string }>()
const { user } = useUser(() => userId)  // Getter function
</script>
```

### Async Composables

```typescript
// ✅ Async composable pattern
export function useFetch<T>(url: MaybeRefOrGetter<string>) {
  const data = ref<T | null>(null)
  const error = ref<Error | null>(null)
  const loading = ref(false)

  const execute = async () => {
    loading.value = true
    error.value = null

    try {
      const response = await fetch(toValue(url))
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`)
      }
      data.value = await response.json()
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  }

  // Auto refetch when reactive URL changes
  watchEffect(() => {
    toValue(url)  // Track dependency
    execute()
  })

  return {
    data: readonly(data),
    error: readonly(error),
    loading: readonly(loading),
    refetch: execute
  }
}

// Usage
const { data, loading, error, refetch } = useFetch<User[]>('/api/users')
```

### Lifecycle and Cleanup

```typescript
// ✅ Properly handle lifecycle in composables
export function useEventListener(
  target: MaybeRefOrGetter<EventTarget>,
  event: string,
  handler: EventListener
) {
  // Add after component mounts
  onMounted(() => {
    toValue(target).addEventListener(event, handler)
  })

  // Remove on component unmount
  onUnmounted(() => {
    toValue(target).removeEventListener(event, handler)
  })
}

// ✅ Use effectScope to manage side effects
export function useFeature() {
  const scope = effectScope()

  scope.run(() => {
    // All reactive effects within this scope
    const state = ref(0)
    watch(state, () => { /* ... */ })
    watchEffect(() => { /* ... */ })
  })

  // Clean up all effects
  onUnmounted(() => scope.stop())

  return { /* ... */ }
}
```

---

## Performance Optimization

### v-memo

```vue
<!-- ✅ v-memo: Cache sub-trees, avoid re-rendering -->
<template>
  <div v-for="item in list" :key="item.id" v-memo="[item.id === selected]">
    <!-- Only re-renders when item.id === selected changes -->
    <ExpensiveComponent :item="item" :selected="item.id === selected" />
  </div>
</template>

<!-- ✅ With v-for -->
<template>
  <div
    v-for="item in list"
    :key="item.id"
    v-memo="[item.name, item.status]"
  >
    <!-- Only re-renders when name or status changes -->
  </div>
</template>
```

### defineAsyncComponent

```vue
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

// ✅ Lazy-load components
const HeavyChart = defineAsyncComponent(() =>
  import('./components/HeavyChart.vue')
)

// ✅ With loading and error states
const AsyncModal = defineAsyncComponent({
  loader: () => import('./components/Modal.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorDisplay,
  delay: 200,  // Delay showing loading (avoid flicker)
  timeout: 3000  // Timeout
})
</script>
```

### KeepAlive

```vue
<template>
  <!-- ✅ Cache dynamic components -->
  <KeepAlive>
    <component :is="currentTab" />
  </KeepAlive>

  <!-- ✅ Specify which components to cache -->
  <KeepAlive include="TabA,TabB">
    <component :is="currentTab" />
  </KeepAlive>

  <!-- ✅ Limit cache count -->
  <KeepAlive :max="10">
    <component :is="currentTab" />
  </KeepAlive>
</template>

<script setup lang="ts">
// KeepAlive component lifecycle hooks
onActivated(() => {
  // Component activated (restored from cache)
  refreshData()
})

onDeactivated(() => {
  // Component deactivated (entering cache)
  pauseTimers()
})
</script>
```

### Virtual Lists

```vue
<!-- ✅ Use virtual scrolling for large lists -->
<script setup lang="ts">
import { useVirtualList } from '@vueuse/core'

const { list, containerProps, wrapperProps } = useVirtualList(
  items,
  { itemHeight: 50 }
)
</script>
<template>
  <div v-bind="containerProps" style="height: 400px; overflow: auto">
    <div v-bind="wrapperProps">
      <div v-for="item in list" :key="item.data.id" style="height: 50px">
        {{ item.data.name }}
      </div>
    </div>
  </div>
</template>
```

---

## Review Checklist

### Reactivity System
- [ ] ref for primitives, reactive for objects (or use ref consistently)
- [ ] No destructuring reactive objects (or used toRefs)
- [ ] Props maintaining reactivity when passed to composables
- [ ] shallowRef/shallowReactive used for large object optimization
- [ ] No side effects in computed

### Props & Emits
- [ ] defineProps uses TypeScript type declarations
- [ ] Complex defaults use withDefaults + factory functions
- [ ] defineEmits has complete type definitions
- [ ] No direct modification of props
- [ ] Consider using defineModel to simplify v-model (Vue 3.4+)

### Vue 3.5 New Features (If Applicable)
- [ ] Using Reactive Props Destructure to simplify props access
- [ ] Using useTemplateRef instead of ref attribute
- [ ] Forms use useId for SSR-safe IDs
- [ ] Using onWatcherCleanup for complex cleanup logic

### Watchers
- [ ] watch/watchEffect have appropriate cleanup functions
- [ ] Async watch handles race conditions
- [ ] flush: 'post' used for DOM-operation watchers
- [ ] Avoid overusing watchers (prefer computed)
- [ ] Consider once: true for one-time listening

### Templates
- [ ] v-for uses unique and stable keys
- [ ] v-if and v-for not on same element
- [ ] Event handlers use methods instead of inline complex logic
- [ ] Large lists use virtual scrolling

### Composables
- [ ] Related logic extracted to composables
- [ ] Composables return reactive refs (not .value)
- [ ] Pure functions not wrapped as composable
- [ ] Side effects cleaned up on component unmount
- [ ] Using effectScope for complex side effects

### Performance
- [ ] Large components split into smaller ones
- [ ] Using defineAsyncComponent for lazy loading
- [ ] Avoid unnecessary reactive conversions
- [ ] v-memo used for expensive list rendering
- [ ] KeepAlive used for caching dynamic components
