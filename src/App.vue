<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

onMounted(async () => {
  await fetchCabinetData();

  // После загрузки данных проверяем, что есть в sessionStorage
  const savedExpandedProcesses = sessionStorage.getItem("expandedProcesses");

  if (savedExpandedProcesses !== null) {
    expandedProcesses.value = JSON.parse(savedExpandedProcesses);
  } else {
    // Если ничего не сохранено — инициализируем в зависимости от allExpanded
    cabinetData.value.forEach((process) => {
      expandedProcesses.value[process.id] = allExpanded.value;
    });
  }

  // Вотчер сохраняет состояние при любом изменении
  watch(
    expandedProcesses,
    (newVal) => {
      sessionStorage.setItem("expandedProcesses", JSON.stringify(newVal));
    },
    { deep: true }
  );

  fetchMaterialsData();
  fetchCommentListData();
  fetchUsersData();
  fetchCoworkers();

  observer = new MutationObserver(async (mutationsList) => {
    await nextTick();

    mutationsList.forEach(() => {
      changeStepperStyle();
      changeFontSizeStyle();
      changeAnotherFontSizeStyle();
      changeLeftLineStyle();
      changeQItemLabelSize();
    });
  });

  observer.observe(document.querySelector(".content-container"), {
    childList: true,
    subtree: true,
    attributes: true,
  });
});
