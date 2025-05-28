<!--Контейнер для делегирования задач-->
<div class="delegate-container" style="margin-top: 15px">
  <q-tree
    :nodes="treeData"
    node-key="id"
    label-key="name"
    tick-strategy="leaf-filtered"
    tickable
    v-model:selected="selectedTaskIds"
    default-expand-all
  />
</div>
import { ref, computed } from 'vue';

// selectedTaskIds уже у тебя есть
const selectedTaskIds = ref([]);

// Преобразует tasks → children (Quasar требует ключ children)
function transformTasksToTree(tasks) {
  return tasks.map(task => ({
    ...task,
    children: task.tasks ? transformTasksToTree(task.tasks) : []
  }));
}

// computed дерево
const treeData = computed(() => {
  if (!currentTaskList.value?.length) return [];
  return transformTasksToTree(currentTaskList.value);
});
{ name: "mentor_tasks", value: JSON.stringify(selectedTaskIds.value) }
