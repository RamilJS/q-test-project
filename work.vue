const taskIdsSet = new Set();
disabledDelegationTasks.value = [];

const collectDisabledTasks = (tasks) => {
  for (const task of tasks) {
    if (task.disabled && !taskIdsSet.has(task.task_id)) {
      taskIdsSet.add(task.task_id);
      disabledDelegationTasks.value.push(task);
    }
    if (task.tasks && task.tasks.length > 0) {
      collectDisabledTasks(task.tasks);
    }
  }
};

// Внутри fetchUsersData, как у тебя сейчас:
for (const user of response.data.results) {
  const taskList = user.mentor?.task_list || [];
  collectDisabledTasks(taskList);
}

const disabledTaskNames = computed(() =>
  disabledDelegationTasks.value.map(task => task.name)
);
