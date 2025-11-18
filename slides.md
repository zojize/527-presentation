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
class: flex flex-col
---

# What Makes E2E Tests Different?

<v-clicks depth="2" class="text-xl" every="2">

- **Highly asynchronous in nature**
  - Tests race against both external network requests (**`Network`** flakiness) and internal UI updates like rendering and animations (**`Async Wait`** flakiness), the #1 root cause [^1].

- **Harder to detect and reproduce**
  - Failures often only appear in the slow, resource-constrained CI pipeline. These **`Environment`**-specific flakes are hard to reproduce locally because of differences in OS, browser version, or screen resolution [^1].

</v-clicks>

<div class="flex-1" />

<style>
li p {
  margin: 0 auto;
}
</style>

[^1]:(Romano et al., 2021) https://doi.org/10.1109/ICSE43902.2021.00141

<!--
We've mostly talked about flaky tests in the unit test world so far in this class, but E2E tests are a fundamentally different. I want to break down the four key characteristics that make them prone to flakiness

[click] First, E2E tests live in a world of unpredictable timing. This isn't just one problem, but two related ones. On one hand, you have external dependencies. The Romano paper calls this Network flakiness, where your test fails because a real backend service is slow or temporarily unavailable.

On the other hand, you have internal asynchronicity. This is the single biggest cause of flaky tests, a category the paper calls Async Wait. It happens when your test script is executing faster than the browser can finish rendering DOM elements, loading data, or playing an animation. Both of these issues boil down to the same thing: a fundamental mismatch in timing between our test and the application.

[click] Second, the test environment itself becomes a primary source of flakiness. This is why we so often hear, 'It works on my machine!' The paper classifies these as Environment-specific issues. A test might fail only on the Linux CI runner, or in a specific browser version, or at a different screen resolution.

This problem is massively amplified by the nature of CI environments. They are often slower and more resource-constrained than our local development machines. This slowness isn't just an inconvenience; it's a catalyst. It takes those timing-related race conditions we just discussed and makes them far more likely to surface as real, intermittent failures. The slow, inconsistent nature of the CI environment is what makes these flakes so difficult to reproduce and debug.
-->

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

<!--

Async Wait is the number one problem, making up nearly half of all E2E flakes. Let's break down what that actually means. At its core, it's a simple race condition: our test script is giving the browser commands faster than the browser can keep up.

This single category can be broken down into three more specific types of race conditions.

[click] [click] [click]

First, we have Network Resource Loading Imagine your test clicks a button that triggers a `fetch` request to an API to load a list of items. Your test script, running at full speed, immediately moves to the next line and asserts that the list contains ten items.

But of course, the network request is still pending. It hasn't even hit the server yet, let alone returned a response. The assertion fails because it's checking for a state that is dependent on a slow, external operation. This is a race between your test script and the network.

[click]

Second, we have Resource Rendering. This one is more subtle. In this case, the data might have already returned from the network and be present in the browser's memory, but it's not yet visible or interactable on the screen.

Think about a modal dialog that fades in with a CSS transition. The DOM element for the modal might exist the instant you click the button, but it's not yet visible or stable. If your test immediately tries to click a button *inside* that modal, it will fail with an 'element not interactable' error. This is a race between your test script and the browser's rendering engine."

[click]

Finally, a particularly tricky subset of rendering issues is Animation Timing. This happens when an element is technically visible but is still moving. A classic example is a notification toast that slides in from the side of the screen.

Your test might correctly wait for it to be visible, but then immediately try to click its 'close' button. If the animation is still in progress, the click might miss or hit the wrong coordinates. This problem is even worse in CI environments, where headless browsers often throttle or change how they handle animations, making the timing completely different from what you see on your local machine.

So, as you can see, this one 'Async Wait' category covers a whole range of timing problems—racing the network, racing the renderer, and racing animations. Now, let's look at what this looks like in a real piece of code and, more importantly, how we fix it.

-->

---

# Fixing Async Flakes

<v-clicks depth="2" every="2" class="text-2xl">

- **1. Add/Modify Wait/Sleep (Most Common)**
  - The most frequent fix was to introduce or change a waiting mechanism. However, *how* you wait is critical.

- **2. Modify Test Logic**
  - Instead of just waiting, developers would change the test's flow to better synchronize with the application's state.

- **3. Modify Assertion**
  - Changing the assertion to wait for an observable outcome rather than checking a state immediately.

</v-clicks>


---
layout: cover
---

# DEMO

<!-- Cypress todolist mvp demo -->


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
import { describe, it, expect, test, beforeEach } from 'vitest'

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

# [`TSV-TOD`](https://github.com/zojize/TSV-TOD)

<p class="text-sm text-gray"> TypeScript Vitest Test Order-dependency Detector (Name inspired by <a href="https://github.com/Negar-Hashemi/JS-TOD">https://github.com/Negar-Hashemi/JS-TOD</a>) </p>

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
layout: cover
---

# DEMO

---
layout: cover
---

# Some Results

---
class: p-4 leading-[0.5rem]
---

| Project                                  | OD Test Files | Failing Test Files | Total Test Files |
| ---------------------------------------- | ------------- | ------------------ | ---------------- |
| rolldown/tsdown                          | 5             | 4                  | 12               |
| antfu/unocss                             | 4             | 4                  | 87               |
| vueuse/vueuse                            | 4             | 27                 | 210              |
| paritytech/asset-transfer-api            | 1             | 7                  | 92               |
| antfu/eslint-config                      | 0             | 2                  | 3                |
| antfu/eslint-plugin-command              | 0             | 0                  | 21               |
| aooiuu/any-reader                        | 0             | 1                  | 2                |
| davelosert/vitest-coverage-report-action | 0             | 0                  | 14               |
| elk-zone/elk                             | 0             | 0                  | 6                |
| eslint-stylistic/eslint-stylistic        | 0             | 1                  | 122              |
| mayneyao/eidos                           | 0             | 7                  | 29               |
| slidevjs/slidev                          | 0             | 0                  | 5                |
| sxzz/unplugin-utils                      | 0             | 0                  | 2                |
| tinylibs/tinyspy                         | 0             | 0                  | 3                |

---

# Causes of OD Flaky Tests

-   **Shared Mutable State:** Tests that modify shared state without proper isolation can lead to order-dependent failures.
-   **Improper Cleanup:** Tests that do not clean up after themselves can leave the environment in an unexpected state for subsequent tests.
-   **Improper Serialization/Deserialization:** Tests that rely on serialization and deserialization snapshots may fail if the order of operations affects the serialized state.

---

# Discussion [^1]

-  Low prevalence of order-dependent tests in JavaScript compared to other languages like Python and Java.
-  Attributed to:
   1. JavaScript developers' test organization practices (grouping related tests together).
   2. Testing frameworks (Vitest, Jest) running tests files in parallel by default and run test cases in the order they appear within files.
   3. Different programming practices (e.g., functional programming style) reducing shared mutable state.
   4. Closure and module scoping limiting unintended interactions between tests.
-  No major language-specific factors causing test order dependency in JavaScript.

[^1]:(Hashemi et al., 2025) https://arxiv.org/abs/2501.12680

---

# Next Steps

- Collect more Vitest projects to analyze.
- Implement classification step to identify NOD/OD tests and vitim/polluter pairs
- Identify specific Victim, Polluter and Cleaners
- Publish as NPM package
- OR PR to Vitest repo


---

# References

- A. Romano, Z. Song, S. Grandhi, W. Yang and W. Wang, "An Empirical Analysis of UI-Based Flaky Tests," 2021 IEEE/ACM 43rd International Conference on Software Engineering (ICSE), Madrid, ES, 2021, pp. 1585-1597, doi: 10.1109/ICSE43902.2021.00141.
- Liu, X., Song, Z., Fang, W., Yang, W., & Wang, W. (2024). WEFix: Intelligent Automatic Generation of Explicit Waits for Efficient Web End-to-End Flaky Tests. ArXiv. https://doi.org/10.1145/3589334.3645628
- Hashemi, N., Tahir, A., Rasheed, S., Shi, A., & Blagojevic, R. (2025). Detecting and Evaluating Order-Dependent Flaky Tests in JavaScript. ArXiv. https://arxiv.org/abs/2501.12680

---