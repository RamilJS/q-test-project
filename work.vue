<q-tab-panels v-model="activeCoworkersTab" animated :keep-alive="false">
  <q-tab-panel name="recommendation">
    <!-- Содержимое вкладки формы рекомендации коллег -->
  </q-tab-panel>
  <q-tab-panel name="list">
    <!-- Содержимое вкладки списка рекомендованных коллег -->
  </q-tab-panel>
</q-tab-panels>

const openCoworkersModal = async (tab = 'recommendation') => {
  isCoworkersModalOpen.value = true;
  await nextTick();

  activeCoworkersTab.value = tab;
  await nextTick();

  if (tab === 'recommendation') {
    await fetchCollaboratorList();
  } else if (tab === 'list') {
    await fetchCoworkers();
  }

  await nextTick();
};

watch(activeCoworkersTab, async (newTab, oldTab) => {
  if (!isCoworkersModalOpen.value || newTab === oldTab) return;

  if (newTab === 'recommendation') {
    await fetchCollaboratorList();
  } else if (newTab === 'list') {
    await fetchCoworkers();
  }
});
