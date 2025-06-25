Нужно дождаться, пока модальное окно полностью откроется, а потом установить activeCoworkersTab. Это делается с помощью nextTick.

const openCoworkersModal = async (tab = 'recommendation') => {
  isCoworkersModalOpen.value = true;

  await nextTick(); // дождаться рендера диалога

  activeCoworkersTab.value = tab;

  if (tab === 'recommendation') {
    await fetchCollaboratorList();
  } else if (tab === 'list') {
    await fetchCoworkers();
  }
};

🚀 Дополнительно: сделать activeCoworkersTab всегда по умолчанию
по умолчанию активировалась вкладка "recommendation", можно просто в watch на isCoworkersModalOpen добавить:
watch(isCoworkersModalOpen, (val) => {
  if (val) {
    activeCoworkersTab.value = 'recommendation';
  }
});

вкладка не отображается корректно, можешь попробовать заменить v-show на v-if в q-tab-panel (реже требуется, но может помочь в сложных случаях рендера).

Пример:
<q-tab-panel
  name="recommendation"
  v-if="activeCoworkersTab === 'recommendation'"
>
