watch(activeCoworkersTab, async (newTab) => {
  if (!isCoworkersModalOpen.value) return;

  if (newTab === 'recommendation') {
    await fetchCollaboratorList();
  } else if (newTab === 'list') {
    await fetchCoworkers();
  }
}, { immediate: true });

const openCoworkersModal = () => {
  activeCoworkersTab.value = 'recommendation'; // сбрасываем каждый раз
  isCoworkersModalOpen.value = true;
};
