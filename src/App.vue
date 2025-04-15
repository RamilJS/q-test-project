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
