---
theme: seriph
title: E2E Flaky Tests
transition: slide-left
duration: 30min
background: '#f0f0f0'
---

# E2E Flaky Tests

Jeff Zou

<PoweredBySlidev />

---

# What Makes E2E Tests Different?

<v-clicks class="text-2xl">

- Require external resources
- Highly asynchronous in nature
- Harder to detect and reproduce
- Slow in CI environments

</v-clicks>

---

<figure>
  <img
    src="/Summary of Root Cause Categories Found.gif"
    alt="Summary of Root Cause Categories Found" />
  <figcaption><a href="https://ieeexplore.ieee.org/abstract/document/9402129#table4">^ Summary of Root Cause Categories Found</a></figcaption>
</figure>

---

# Async Await Flaky Tests (45%)

<v-clicks depth="2">

-   **Problem:** Test script executes an action or assertion before the browser is ready.
-   **Why?**
    1.  **Network Resource Loading:** Fetching data to populate a list, submitting a form, etc.
    2.  **Resource Rendering:** Waiting for CSS animations/transitions, for an element to become visible and stable, or for a virtual DOM to patch the real DOM.
    3.  **Animation Timing Issue:** Flaky tests relying on animations are sensitive to timing differences in the running environment and may be heavily optimized to skip animation events.

</v-clicks>


---

# TODO: more slides on E2E flaky tests and demo


---
layout: image-right
image: https://vitest.dev/logo-shadow.svg
backgroundSize: contain
class: flex items-center
---

# Vitest OD Tests

---

# What is Vitest?

- A blazing fast unit test framework for JavaScript/TypeScript powered by Vite
- Compatible with Jest API, which used to be the de facto standard for JS/TS testing
- Smart & instant watch mode
- Out-of-box ESM, TypeScript and JSX support

<div class="mt-8 flex space-x-8">
  <img src="https://vite.dev/logo.svg" class="w-32 h-32" alt="Vite Logo" />
  <img src="https://github.com/jestjs/jest/blob/main/website/static/img/jest.png?raw=true" class="w-32 h-32" alt="Jest Logo" />
  <img src="https://github.com/microsoft/TypeScript-Website/blob/v2/packages/typescriptlang-org/static/branding/ts-logo-512.png?raw=true" class="w-32 h-32" alt="TypeScript Logo" />
</div>

---
class: flex flex-col
---
# Example Test File

<v-switch>

<template #0>

This is a simple test file that tests array operations using Vitest.

</template>

<template #1>

Each `describe` function groups related tests together and creates a context for tests to run in, they can also be nested to create a hierarchy of test contexts.

</template>

<template #2>

The `it`/`test` functions define individual test cases that contain assertions to verify the expected behavior of the code being tested.

</template>

<template #3>

The `beforeEach` function registers a callback to be called before each of the tests in the current context runs, allowing for setup code to be shared across multiple tests. There are other lifecycle hooks like `afterEach`, `beforeAll`, and `afterAll` as well.

</template>

</v-switch>

```ts {all|3,10,22|11-14,16-19,23-27,29-32|6-8}{lines:true, maxHeight:'75%', at:1}
import { describe, it, expect, test } from 'vitest'

describe('Array operations', () => {
  let arr: number[]

  beforeEach(() => {
    arr = [1, 2, 3]
  })

  describe('push method', () => {
    it('should add an element to the end of the array', () => {
      arr.push(4)
      expect(arr).toEqual([1, 2, 3, 4])
    })

    it('should increase the length of the array', () => {
      arr.push(5)
      expect(arr.length).toBe(4)
    })
  })

  describe('pop method', () => {
    test('should remove the last element from the array', () => {
      const lastElement = arr.pop()
      expect(lastElement).toBe(3)
      expect(arr).toEqual([1, 2])
    })

    test('should decrease the length of the array', () => {
      arr.pop()
      expect(arr.length).toBe(2)
    })
  })
})
```

---

# Concurrency and Test Grouping

<v-switch>

<template #0>

Internally Vitest treats `describe` and `it/test` blocks as tasks and groups all tasks in the same context into the same task group by default. Unless explicitly annotated, all tasks are run sequentially within each group.

</template>

<template #1>

With explicit `concurrent/sequential` annotations, tasks are further grouped into concurrent/sequential task groups. Which means all tasks in a concurrent group can run in parallel with other concurrent groups if `sequence.concurrent` is enabled, while tasks in a sequential group will still run one after another.

</template>

</v-switch>

````md magic-move {lines:true, at:1}
```ts {4-7}
import { describe, it, expect, test } from 'vitest'
describe('Array operations', () => {
  let arr = [1, 2, 3]
  it('should be an array'  , () => { /* ... */ })
  it('should have length 3', () => { /* ... */ })
  describe('push method'   , () => { /* ... */ })
  describe('pop method'    , () => { /* ... */ })
})
```
```ts {4-7}
import { describe, it, expect, test } from 'vitest'
describe('Array operations', () => {
  let arr = [1, 2, 3]
  it.concurrent('should be an array'  , () => { /* ... */ }) // Group 1
  it.concurrent('should have length 3', () => { /* ... */ }) // Group 1
  describe.sequential('push method'   , () => { /* ... */ }) // Group 2
  describe.sequential('pop method'    , () => { /* ... */ }) // Group 2
})
```
````

---

# `TSV-TOD`

<p class="text-sm text-gray"> TypeScript Vitest Test Order-dependency Detector (Name inspired by <a href="https://github.com/Negar-Hashemi/JS-TOD">`JS-TOD`</a>) </p>

- Custom Vitest sequencer that permutes test order with seeded runs
- Supports iDFlakies-inspired configurations: `original`, `random-group`/`test`, `reverse-group`/`test`
- Respects Vitest's sequential/concurrent task grouping when shuffling, does not intersperse tests across groups
- CLI tool to run Vitest projects with specified order configurations and collect results
- Surfaces order-dependent flakes by rerunning files, logging failing orders and seeds

---

# Random Configurations

<v-switch>

<template #0>

-   **original:** Run tests in the original order as defined in the test files.

```ts
import { describe, it, expect } from 'vitest'

describe('spam', () => {
  it('foo', () => { /* ... */ })
  it('bar', () => { /* ... */ })
  describe('ham', () => {
    it('baz', () => { /* ... */ })
    it('qux', () => { /* ... */ })
  })
  describe('eggs', () => {
    it('abc', () => { /* ... */ })
    it('xyz', () => { /* ... */ })
  })
})
```

</template>

<template #1>

-   **random-group:** Randomize the order of test groups (describe blocks) while keeping the order of tests within each group intact.

<Suspense>
  <RandomConfigurationExample order="random-group"/>
</Suspense>

</template>

<template #2>

-   **random-test:** Randomize the order of individual tests (it/test blocks) within their respective groups in addition to randomizing the order of test groups.

<Suspense>
  <RandomConfigurationExample order="random-test"/>
</Suspense>

</template>

<template #3>

-   **reverse-test:** Reverse the order of individual tests (it/test blocks) within their respective groups while keeping the order of test groups intact.

<Suspense>
  <RandomConfigurationExample order="reverse-test"/>
</Suspense>

</template>

<template #4>

-   **reverse-group:** Reverse the order of test groups (describe blocks) while keeping the order of tests within each group intact.

<Suspense>
  <RandomConfigurationExample order="reverse-group"/>
</Suspense>

</template>

</v-switch>

---

# TODO: slides on results, analysis and next Steps