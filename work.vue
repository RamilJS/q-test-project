openCoworkersModal необязательный параметр

const openCoworkersModal = async (defaultTab = null) => {
  // если передали вкладку — принудительно её выставляем
  if (defaultTab) {
    activeCoworkersTab.value = defaultTab;
  }

  isCoworkersModalOpen.value = true;

  if (activeCoworkersTab.value === 'list') {
    await fetchCoworkers();
  } else if (activeCoworkersTab.value === 'recommendation') {
    await fetchCollaboratorList();
  }
};

Кнопка «Порекомендовать ещё коллегу»
Оставь как было по смыслу, но вызывай с нужной вкладкой
const recommendAnotherCoworker = async () => {
  isCoworkersSuccessModalOpen.value = false;
  await nextTick();
  await openCoworkersModal('recommendation'); // всегда форма
};

Кнопка «Закрыть» в модалке успеха
Задай «дефолт» для следующего открытия — список:
const closeSuccessModal = async () => {
  isCoworkersSuccessModalOpen.value = false;
  // на следующее открытие основного окна — список
  activeCoworkersTab.value = 'list';
  await nextTick();
};

Кнопка «Рекомендовать коллег» в главном интерфейсе
Там, где открываешь основное модальное окно снаружи (на странице), вызывай:
<q-btn ... @click="openCoworkersModal('list')" />
