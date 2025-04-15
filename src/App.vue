<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

const copyEmployeeStatuses = () => {
  cabinetData.value = cabinetData.value.map(task => ({
    ...task,
    completedMentor: task.completed
  }));

  showToast("Статусы скопированы");
};

const copyEmployeeStatuses = async () => {
  for (const task of cabinetData.value) {
    // Только если значения отличаются, чтобы не дергать лишние запросы
    if (task.completed !== task.completedMentor) {
      await handleToggle(task.completed, task.id, task); // здесь `task` — это и есть "month", тк нет структуры месяцев
    }
  }

  showToast("Статусы скопированы");
};
