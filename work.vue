const recommendAnotherCoworker = async () => {
  isCoworkersSuccessModalOpen.value = false;

  // Устанавливаем нужную вкладку
  activeCoworkersTab.value = 'recommendation';

  await nextTick(); // чтобы гарантировать закрытие предыдущего окна

  // Открываем модалку уже с нужной вкладкой
  await openCoworkersModal();
};


const openCoworkersModal = async () => {
  isCoworkersModalOpen.value = true;

  // Не сбрасываем вкладку — она уже выбрана ранее в recommendAnotherCoworker

  if (activeCoworkersTab.value === 'list') {
    await fetchCoworkers();
  } else if (activeCoworkersTab.value === 'recommendation') {
    await fetchCollaboratorList();
  }
};
