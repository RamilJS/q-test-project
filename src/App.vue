<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

const copyEmployeeStatuses = async () => {
  const updates = [];

  for (const month of cabinetData.value) {
    for (const task of month.tasks) {
      if (task.completedMentor !== task.completed) {
        task.completedMentor = task.completed;
        updates.push(handleToggle(task.completed, task.id, month));
      }
    }
  }

  await Promise.all(updates);
  showToast("Статусы скопированы");
};
