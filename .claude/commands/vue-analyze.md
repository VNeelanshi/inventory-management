---
name: vue-analyze
description: Analyze Vue component structure and suggest optimizations for performance and code reuse
---

Analyze Vue component(s) for performance issues and code reuse opportunities.

## Target

If a file path argument was provided (e.g. `/vue-analyze client/src/views/Orders.vue`), analyze only that file.
Otherwise, use Glob to find all `*.vue` files under `client/src/` and analyze them all.

## Analysis Steps

### 1. Read each component

Use the Read tool to load each `.vue` file. For each component, extract:

- The `<template>` block
- The `<script setup>` (or `<script>`) block
- The `<style>` block

### 2. Performance audit

Check for each of the following and record every finding with the file path and line number:

**Reactivity / computed**

- Logic in the template that should be a `computed` (e.g. filtered arrays, formatted strings, conditional class lists built inline)
- `watch` used where `computed` would suffice (watch that only sets another ref)
- Computed properties that depend on the entire reactive object when only one field is needed

**Template rendering cost**

- `v-for` without a `:key`, or using array index as `:key` on a list that can reorder
- `v-if` / `v-else` blocks on elements that toggle frequently — suggest `v-show` if the DOM is expensive to recreate
- Method calls directly in the template (e.g. `:value="formatPrice(item.cost)"` inside `v-for`) that re-run on every render — suggest `computed` or moving the call outside the loop
- Deep object mutations that bypass Vue reactivity (direct index assignment, `delete obj.key`)

**Component size and responsibility**

- Single-file components longer than ~300 lines of template or ~200 lines of script — suggest extraction candidates
- Multiple distinct visual sections in one template that have no shared state — suggest splitting into child components

**API / async patterns**

- `async` calls not wrapped in error handling inside `onMounted` or watchers
- Redundant API calls: the same endpoint fetched in sibling components when a shared composable or parent prop would suffice

### 3. Code reuse audit

**Composable extraction**

- Logic blocks (fetch + loading + error state, filter state + reset, pagination) repeated across two or more components — propose a `use<Name>.js` composable and show the interface

**Component extraction**

- Repeated template fragments (same card structure, same table row pattern, same form field group) across files — propose a new shared component, name it, and show the required props

**Props / emits hygiene**

- Props that are passed straight through to a child without transformation — consider whether the grandchild should receive them directly or a slot would be cleaner
- Events emitted with no payload when the parent already has access to the reactive state

**Style duplication**

- Identical CSS rules duplicated across `<style>` blocks — suggest moving to the global stylesheet in `App.vue` or a shared CSS custom property

### 4. Vue 3 / Composition API best practices

- `<script setup>` not used where it could replace Options API or the old `setup()` return object
- `reactive()` used for primitive values (should be `ref()`)
- Destructuring a `reactive()` object without `toRefs()` (breaks reactivity)
- `$refs` used in template when a `ref()` in `<script setup>` is cleaner
- Missing `onUnmounted` cleanup for event listeners, timers, or subscriptions set up in `onMounted`

### 5. Report

Output a structured Markdown report with these sections:

```
## Vue Component Analysis

### Summary
<N> components analyzed. <X> performance findings, <Y> reuse opportunities.

### Performance Findings
For each finding:
- **[SEVERITY: high/medium/low]** `path/to/Component.vue:<line>` — description of issue
  - Suggestion: concrete fix with a short code snippet if helpful

### Code Reuse Opportunities
For each opportunity:
- **`path/to/Component.vue`** — description of duplication or extraction candidate
  - Recommendation: proposed composable name / component name, props interface, rough implementation sketch

### Quick Wins (fix in < 5 min)
Bullet list of the easiest fixes sorted by impact.

### Larger Refactors
Bullet list of extractions that need more planning, with effort estimate (S/M/L).
```

Be specific — include file paths, line numbers, and short before/after code snippets for every finding. Do not invent issues; only report what is actually present in the code.
