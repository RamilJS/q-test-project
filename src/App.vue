
const selectedTaskIds = ref([]); // <--- добавить это

function transformTasksToTree(tasks) {
  return tasks.map(task => {
    if (task.checked) {
      selectedTaskIds.value.push(task.id); // добавляем в ticked список
    }

    return {
      ...task,
      children: task.tasks ? transformTasksToTree(task.tasks) : []
    };
  });
}

  const treeData = computed(() => {
  if (!currentTaskList.value?.length) return [];
  selectedTaskIds.value = []; // очищаем перед каждой трансформацией
  return transformTasksToTree(currentTaskList.value);
});

<q-tree
  :nodes="treeData"
  node-key="id"
  label-key="name"
  tick-strategy="leaf"
  tickable
  v-model:ticked="selectedTaskIds"
  default-expand-all
/>

  
