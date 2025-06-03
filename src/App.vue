
<template>
  <div class="delegate-container" style="margin-top: 15px">
    <q-tree
      :nodes="treeData"
      node-key="id"
      label-key="name"
      tick-strategy="leaf"
      tickable
      v-model:ticked="selectedTaskIds"
      default-expand-all
    />
  </div>
</template>

<script setup>
import { ref, watch } from 'vue'
import { QTree } from 'quasar'

// Пример: данные от сервера (замени на реальные)
const currentTaskList = ref([
  {
    id: 'all',
    name: 'Делегировать все задачи',
    checked: true,
    disabled: false,
    tasks: [
      {
        id: 'task1',
        name: 'Подготовка к выходу нового сотрудника',
        checked: false,
        disabled: false,
        tasks: [
          {
            id: 'subtask1',
            name: 'Подготовьте заявку на пропуск',
            checked: true,
            disabled: false,
            tasks: []
          }
        ]
      },
      {
        id: 'task2',
        name: 'Знакомство с командой',
        checked: true,
        disabled: false,
        tasks: []
      }
    ]
  }
])

const selectedTaskIds = ref([])
const treeData = ref([])

// Рекурсивная функция преобразования задач в структуру для q-tree
function transformTasksToTree(tasks) {
  return tasks.map(task => ({
    ...task,
    children: task.tasks ? transformTasksToTree(task.tasks) : []
  }))
}

// Рекурсивный сбор id отмеченных чекбоксов
function collectCheckedIds(tasks, result = []) {
  for (const task of tasks) {
    if (task.checked) result.push(task.id)
    if (task.tasks && task.tasks.length > 0) {
      collectCheckedIds(task.tasks, result)
    }
  }
  return result
}

// Следим за обновлением данных от сервера
watch(currentTaskList, (newTasks) => {
  if (!newTasks || !newTasks.length) {
    treeData.value = []
    selectedTaskIds.value = []
    return
  }

  treeData.value = transformTasksToTree(newTasks)
  selectedTaskIds.value = collectCheckedIds(newTasks)
}, { immediate: true })
</script>
