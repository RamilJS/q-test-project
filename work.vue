<q-dialog v-model="isCoworkersModalOpen">
  <q-card style="min-width: 800px;">
    <!-- Заголовок модального окна с вкладками -->
    <q-card-section class="row" style="justify-content: space-between; padding-bottom: 0;">
      <q-tabs
        v-model="activeCoworkersTab"
        dense
        class="text-grey"
        active-color="primary"
        indicator-color="primary"
        align="left"
      >
        <q-tab name="recommendation" label="Форма рекомендации коллег" />
        <q-tab name="list" label="Список рекомендованных коллег" />
      </q-tabs>
      <q-btn
        flat
        icon="fa-solid fa-xmark"
        color="primary"
        v-close-popup
      />
    </q-card-section>

    <!-- Исправленная секция с содержимым -->
    <q-tab-panels v-model="activeCoworkersTab" animated>
      <!-- Вкладка формы рекомендации -->
      <q-tab-panel name="recommendation">
        <div class="text-h6 text-bold">Порекомендовать коллег</div>
        <!-- ... остальное содержимое формы ... -->
      </q-tab-panel>

      <!-- Вкладка списка рекомендованных -->
      <q-tab-panel name="list">
        <div class="text-h6">Рекомендованные коллеги</div>
        <!-- ... остальное содержимое списка ... -->
      </q-tab-panel>
    </q-tab-panels>
  </q-card>
</q-dialog>

watch(activeCoworkersTab, async (newTab) => {
  if (!isCoworkersModalOpen.value) return;
  
  if (newTab === 'recommendation') {
    await fetchCollaboratorList();
  } else if (newTab === 'list') {
    await fetchCoworkers();
  }
}, { immediate: true }); // Добавьте immediate:true для загрузки при открытии

<q-tab-panels v-model="activeCoworkersTab" animated keep-alive>
