---
# You can also start simply with 'default'
theme: geist
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
# background: https://cover.sli.dev
# some information about your slides (markdown enabled)
# apply unocss classes to the current slide
# class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# open graph
# seoMeta:
#  ogImage: https://cover.sli.dev
layout: cover
---

# Single source of truth & Vue

## Jari Zwarts

---
layout: two-cols-header
---

# two-way bindings

Say, we have a parent component that has a `count` variable, and we want to pass it to a child component and have it update the parent when it changes.  
Unfortunately, I have seen this being handled too often in the following (simplified) way:

::left::
```vue [parent.vue] {2-3,6}
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
</script>
<template>
  <Child v-model="count" />
</template>
```

::right::
```vue [child.vue] {2-12,16}
<script setup lang="ts">
const count = ref(0);
const emit = defineEmits<{
  (e: 'update:modelValue', value: number): void;
}>();
const props = defineProps<{
  modelValue: number;
}>();
watch(count, (newValue) => {
  // emit an event to update the parent
  emit('update:modelValue', newValue);
});
</script>

<template>
{{ count }}
</template>
```

---

# the problem

This _technically_ works, but has a few subtle, but major problems:  

- **Complexity**: You now have to keep track of two states, which can lead to bugs and confusion.
- **Maintainability**: If you want to change the way the state is managed, you have to change both the parent and the child components.
- **Infinite loops**: It's very easy to now get stuck into infinite update loops where: child <-> parent and parent <-> child.
- **Performance**: You have to manually update the parent state every time the child's state changes, which can lead to unnecessary re-renders.

---
layout: quote
---

# watch

The first line that the vue docs mention for `watch`, is that it's meant for _side effects_:

_Computed properties allow us to declaratively compute derived values. However, there are cases where we need to perform "side effects" in reaction to state changes - for example, mutating the DOM, or changing another piece of state based on the result of an async operation._

<!-- it's basically useEffect from react -->
---

# watch: what are side effects?

Syncing your state to that of an external one, e.g.

- The result of a async operation (e.g. fetching data from somewhere)
- A third party library (e.g. sending an analytics event)
- Manual DOM mutation (e.g. updating the window title)

Summed up, side effects are anything **that is not a pure function of the state**.  

<!-- 
What are side effects?
Vue docs: "for example, mutating the DOM, or changing another piece of state based on the result of an async operation.""
React docs (useEffect): "For example, you might want to control a non-React component based on the React state, set up a server connection, or send an analytics log when a component appears on the screen."
-->


---
layout: two-cols-header
---

# defineModel

`defineModel` is a _macro_ that does the following:

- adds a `modelValue` prop to the component
- emits `update:modelValue` event when the value changes

Notably, it **does not create a copy of the state**. It uses the `modelValue` prop directly while giving you the same experience as a `ref`, so you don't have to worry about keeping two states in sync.

::left::
```vue [parent.vue] {2,3,7}
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
</script>

<template>
  <Child :v-model="count" />
</template>
```

::right::
```vue [child-component.vue] {2,3,7}
<script setup lang="ts">
import { defineModel } from 'vue'
const count = defineModel<number>();
</script>

<template>
  <button @click="count++">{{count}}</button>
</template>
```


---

# defineModel: you already know this.

**defineModel should already your standard way of doing two way bindings**.

_But_, is it really though?  

I've noticed that once passed down model values **get more complex**, we tend to easily pick the duplicated state approach, introducing all the problems we just discussed.

So, let's get **more complex**!

---

# computed: example

Let's say, we have a `pagination` object with all sorts of fields:
```ts
{
  currentPage: number,
  amountPerPage: number
  ...
}
```

And say, we have a child that doesn't care about anything else but `currentPage`, understandably we want to seperate `currentPage` from the pagination object, but still keep reading/writing to the real `pagination` object to prevent having to sync it with `watch` manually.

---

# computed: get/set

```ts
const currentPage = computed({
  get () {
    return pagination.value.currentPage;
  },
  set (value) {
    pagination.value.currentPage = value;
  }
})
```

```vue [parent.vue]
<Child v-model="currentPage" />
```

The child can now stay 'dumb' and only needs to worry about `currentPage`
```vue [child.vue] {2,5}
<script>
const currentPage = defineModel<number>()
</script>
<template>
<button @click="currentPage++">{{currentPage}</button>
</template>
```

<!-- 

-->

---

# computed: transformations

Another example where you might be tempted to (ab)use `watch` is one where you need to do some kind of transformation over the incoming/outgoing data.
Grabbing the pagination example from earlier, what if our `currentPage` is derived from something else?

```ts [pagination.ts]
const pagination = ref({
  amountPerPage: 15,
  offset: 30
})
```

This is also solvable by using computed with get/set:

```ts {3,6}
const currentPage = computed(() => {
  get () {
    return Math.ceil(pagination.value.offset / pagination.value.amountPerPage) + 1; // -> 3
  }
  set (value) {
    pagination.value.offset = (value - 1) * pagination.value.amountPerPage; // -> 30
  }
})
```

---

# computed: lots of boilerplate!

This does end up being a lot of boilerplate, but we can fairly easily abstract this away into a composable:

```ts [useComputedPagination.ts]
function useComputedPagination(pagination: Ref<Pagination>) {
  const currentPage = computed({
    ...
  })
  const amountPerPage = computed({
    ...
  })

  return {
    currentPage,
    amountPerPage,
  }
}
```

---

# computed: lazily evaluated

Calling the composable at different locations in the application (e.g. you might decide to pass the entire pagination object to your child components instead) is fine, `computed` is lazily evaluated, meaning that if we don't read/set it's value, the computations aren't executed.


```ts [some-child-somewhere.vue]
// I don't care for amountPerPage, so I don't use it.
const { currentPage } = useComputedPagination(pagination)
```

---

# conclusion

**Always derive your state!**

- Use `defineModel` when doing two way bindings.
- Use `computed` with `get` (and optionally `set`) to grab/set fields that...
  - may lie deeper
  - needs transformations
- Use `watch` for side effects only. Not for managing state!
- Avoid using `ref` for state that already exists somewhere else!

---
layout: end
---

# fin

- Suggestions?
- Questions?