<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

onMounted(() => {
  fetchCabinetData();
  fetchMaterialsData();
  fetchCommentListData();
  fetchUsersData();
  fetchCoworkers();

  // Восстановление состояния раскрытых процессов
  const savedExpandedProcesses = sessionStorage.getItem('expandedProcesses');
  if (savedExpandedProcesses !== null) {
    expandedProcesses.value = JSON.parse(savedExpandedProcesses);
  } else {
    // Если нет сохраненного состояния, используем allExpanded как дефолт
    cabinetData.value.forEach((process) => {
      expandedProcesses.value[process.id] = allExpanded.value;
    });
  }

  nextTick(() => {
    changeStyles(); // Функция, объединяющая стили
  });

  observer = new MutationObserver(async (mutationsList) => {
    await nextTick();
    changeStyles();
  });

  observer.observe(document.querySelector(".content-container"), {
    childList: true,
    subtree: true,
    attributes: true,
  });
});

// Вотчер на expandedProcesses
watch(
  expandedProcesses,
  (newVal) => {
    sessionStorage.setItem('expandedProcesses', JSON.stringify(newVal));
  },
  { deep: true }
);


