const isAllExpanded = computed(() => {
  const ids = cabinetData.value.map((p) => p.id);
  return ids.length > 0 && ids.every((id) => expandedProcesses.value[id]);
});

const toggleProcessList = () => {
  const newState = !isAllExpanded.value;
  cabinetData.value.forEach((process) => {
    expandedProcesses.value[process.id] = newState;
  });
  saveExpandedStates();
};

const toggleProcess = (processId) => {
  expandedProcesses.value[processId] = !expandedProcesses.value[processId];
  saveExpandedStates();
};

<q-btn
  flat
  color="primary"
  @click="toggleProcessList"
  :label="isAllExpanded ? 'Свернуть все' : 'Раскрыть все'"
/>

return {
  activeTab,
  toastVisible,
  toastMessage,
  expandedProcesses,
  toggleProcessList,
  toggleProcess,
  isAllExpanded,
  // другие переменные
};
