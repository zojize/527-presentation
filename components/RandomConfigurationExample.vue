<script setup lang="ts">
import { createHighlighter } from 'shiki'
import { ShikiMagicMove } from 'shiki-magic-move/vue'
import { computed, ref, nextTick, defineProps } from 'vue'
import { useDarkMode } from '@slidev/client'

const { order = 'random-test' } = defineProps<{
  order?: TestOrder
}>()


const { isDark } = useDarkMode()

const theme = computed(() => isDark.value ? 'vitesse-dark' : 'vitesse-light')

const highlighter = await createHighlighter({
  themes: ['vitesse-light', 'vitesse-dark'],
  langs: ['javascript', 'typescript'],
})

interface Suite {
  type: "suite"
  name: string
  tasks: Array<Task>
  concurrent?: boolean
}

interface Test {
  type: "test"
  name: string
  concurrent?: boolean
}

type Task = Suite | Test

// https://github.com/vitest-dev/vitest/blob/2e7b2b8b98dafc047a3bf2fc0422076ca5e346fa/packages/runner/src/utils/suite.ts#L6-L23
function partitionSuiteChildren(suite: Suite): Array<Array<Task>> {
  let tasksGroup: Array<Task> = []
  const tasksGroups: Array<Array<Task>> = []
  for (const c of suite.tasks) {
    if (tasksGroup.length === 0 || !!c.concurrent === !!tasksGroup[0].concurrent) {
      tasksGroup.push(c)
    } else {
      tasksGroups.push(tasksGroup)
      tasksGroup = [c]
    }
  }
  if (tasksGroup.length > 0) {
    tasksGroups.push(tasksGroup)
  }

  return tasksGroups
}

// https://github.com/vitest-dev/vitest/blob/main/packages/utils/src/random.ts
let seed = +Math.random().toString().slice(2)

function random() {
  const x = Math.sin(seed++) * 10000
  return x - Math.floor(x)
}

function shuffle<T>(array: Array<T>): Array<T> {
  let length = array.length
  const result = array.slice()

  while (length) {
    const index = Math.floor(random() * length--)
    const previous = result[length]
    result[length] = result[index]
    result[index] = previous
    ++seed
  }

  return result
}

export type TestOrder = "original" | "random-group" | "random-test" | "reverse-group" | "reverse-test"

function shuffleTaskGroups(array: Array<Array<Task>>, order: TestOrder) {
  switch (order) {
    case "original":
      return array
    case "random-group":
      return shuffle(array)
    case "random-test":
      return shuffle(array.map((group) => shuffle(group)))
    case "reverse-group":
      return array.slice().reverse()
    case "reverse-test":
      return array.map((group) => group.slice().reverse()).reverse()
  }
}

function shuffleSuite(suite: Suite, order: TestOrder) {
  if (order === "original") {
    return
  }

  const taskGroups = partitionSuiteChildren(suite)
  const shuffled = shuffleTaskGroups(taskGroups, order)
  const flattened = shuffled.flat()
  for (const task of flattened) {
    if (task.type === "suite") {
      shuffleSuite(task, order)
    }
  }
  suite.tasks = flattened
}

const suiteA = ref<Suite>({
  type: "suite",
  name: "spam",
  tasks: [
    { type: "test", name: "foo", concurrent: order.includes('group') },
    { type: "test", name: "bar", concurrent: order.includes('group') },
    {
      type: "suite",
      name: "ham",
      tasks: [
        { type: "test", name: "baz" },
        { type: "test", name: "qux" },
      ],
    },
    {
      type: "suite",
      name: "eggs",
      tasks: [
        { type: "test", name: "abc" },
        { type: "test", name: "xyz" },
      ],
    },
  ],
})

shuffleSuite(suiteA.value, order)

const code = computed(() => {
  return `import { describe, it, expect } from 'vitest'

${generateCode(suiteA.value)}`
})

function generateCode(suite: Suite, indent = 0): string {
  const indentation = '  '.repeat(indent)
  let result = `${indentation}describe('${suite.name}', () => {\n`
  for (const group of partitionSuiteChildren(suite)) {
    for (const task of group) {
      if (task.type === "test") {
        result += `${indentation}  it${task.concurrent ? '.concurrent' : ''}('${task.name}', () => { /* ... */ })\n`
      } else if (task.type === "suite") {
        result += generateCode(task, indent + 1)
      }
    }
  }
  result += `${indentation}})\n`
  return result
}

const seedRenderKey = ref(0)
const refresh = () => {
  seed = +Math.random().toString().slice(2)
  shuffleSuite(suiteA.value, order)
  seedRenderKey.value++
}
</script>

<template>
  <div class="slidev-code-wrapper slidev-code-magic-move relative group">
    <ShikiMagicMove
      class="slidev-code relative shiki overflow-visible"
      lang="ts"
      :theme="theme"
      :highlighter
      :code
      :options="{ 
        containerStyle: false,
        duration: 500,
        stagger: 1,
        delayEnter: 0,
        delayMove: 0,
      }"
    />
    <template v-if="order.startsWith('random')">
      <button
        class="slidev-code-copy absolute top-0 right-0 transition opacity-0 group-hover:opacity-20 hover:!opacity-100"
        title="refresh"
        @click="refresh"
      >
        <ph-arrow-clockwise class="p-2 w-8 h-8" />
      </button>
      <code :key="seedRenderKey">seed: {{ seed }}</code>
    </template>
  </div>
</template>

<style>
.slidev-code-magic-move .shiki-magic-move-enter-from,
.slidev-code-magic-move .shiki-magic-move-leave-to {
  opacity: 0;
}
</style>
