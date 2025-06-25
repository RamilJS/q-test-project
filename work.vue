<q-dialog v-model="isCoworkersModalOpen">
  <q-card style="min-width: 800px;">
    <!-- Заголовок с вкладками -->
    <q-tabs
      v-model="activeCoworkersTab"
      dense
      active-color="primary"
      indicator-color="primary"
      align="left"
      class="q-px-md q-pt-md"
    >
      <q-tab name="recommendation" label="Форма рекомендации коллег" />
      <q-tab name="list" label="Список рекомендованных коллег" />
      <q-space />
      <q-btn flat icon="fa-solid fa-xmark" color="primary" v-close-popup />
    </q-tabs>

    <q-separator />

    <!-- Контейнер для содержимого вкладок -->
    <q-tab-panels 
      v-model="activeCoworkersTab" 
      animated
      class="q-pa-none"
    >
      <!-- Вкладка формы рекомендации -->
      <q-tab-panel name="recommendation" class="q-pa-none">
        <q-card-section>
          <div class="text-h6 text-bold q-mb-md">Порекомендовать коллег</div>
          
          <!-- Имя сотрудника -->
          <div class="text-info-item column q-mb-md">
            <p class="text-info-title text-grey q-mb-xs">Сотрудник</p>
            <p class="text-info-value text-black">{{ personFullname }}</p>
          </div>

          <!-- Поле поиска коллег -->
          <div class="employee-search-container q-mb-md">
            <span class="text-bold">Рекомендации коллег</span>
            <div class="row q-mt-sm">
              <q-input
                v-model="collaboratorSearch"
                @keyup.enter="fetchCollaboratorSearch"
                dense
                outlined
                placeholder="Поиск"
                class="bg-white col"
              />
              <q-btn
                icon="search"
                color="primary"
                flat
                @click="fetchCollaboratorSearch"
              />
            </div>
          </div>

          <!-- Список рекомендованных коллег -->
          <div class="q-pa-sm">
            <div
              v-for="collaborator in collaboratorListData"
              :key="collaborator.id"
              class="q-pa-sm q-mb-sm q-border q-rounded bg-white row items-center"
            >
              <q-radio
                v-model="selectedCoworkerId"
                :val="collaborator.id"
                color="primary"
                class="q-mr-md"
              />
              <div>
                <div><strong>{{ collaborator.fullname }}</strong></div>
                <div>{{ collaborator.position_name }}</div>
                <div>{{ collaborator.subdivision_name }}</div>
              </div>
            </div>

            <div class="row justify-end q-mt-md q-gutter-sm">
              <q-btn
                label="Назад"
                color="primary"
                outline
                :disable="collaboratorPage <= 1"
                @click="prevCollaboratorPage"
              />
              <q-btn
                label="Вперёд"
                color="primary"
                outline
                @click="nextCollaboratorPage"
              />
            </div>
          </div>

          <!-- Комментарий -->
          <div class="q-mt-md">
            <span class="text-grey">Комментарий</span>
            <q-input
              v-model="newComment.text"
              label="Введите описание"
              filled
              clearable
              type="textarea"
              autogrow
              :maxlength="1000"
              class="q-mt-sm q-mb-xs"
            />
            <span class="text-grey text-caption">
              Опишите по какому вопросу будет полезен каждый рекомендованный коллега
            </span>
          </div>
        </q-card-section>

        <q-card-actions align="right" class="q-pa-md">
          <q-btn
            label="Отмена"
            color="white"
            text-color="primary"
            class="q-mr-sm"
            v-close-popup
          />
          <q-btn
            label="Порекомендовать коллег"
            color="primary"
            @click="saveCoworkersRecommendation()"
          />
        </q-card-actions>
      </q-tab-panel>

      <!-- Вкладка списка рекомендованных -->
      <q-tab-panel name="list" class="q-pa-none">
        <q-card-section>
          <div class="text-h6 q-mb-md">Рекомендованные коллеги</div>
          
          <ul class="coworkers-list q-pa-none">
            <li
              v-for="coworkerItem in coworkersListData"
              :key="coworkerItem.id"
              class="coworker-item row q-mb-md"
            >
              <!-- Аватар -->
              <div class="coworker-avatar q-mr-md">
                <q-avatar size="100px">
                  <img 
                    v-if="coworkerItem.person_icon_url"
                    :src="coworkerItem.person_icon_url"
                    style="object-fit: cover;"
                  />
                  <img v-else src="../images/person-default.png" />
                </q-avatar>
              </div>

              <!-- Информация -->
              <div class="column q-gutter-xs">
                <div class="text-black text-weight-bold">{{ coworkerItem.fullname }}</div>
                <div>{{ coworkerItem.position_name }}</div>
                <div>{{ coworkerItem.subdivision_name }}</div>
                <div>{{ coworkerItem.email }}</div>
                <div v-if="coworkerItem.phone">{{ coworkerItem.phone }}</div>
                
                <!-- Комментарий -->
                <div class="q-mt-sm">
                  <div class="text-black">Комментарий:</div>
                  <div>{{ coworkerItem.comment }}</div>
                </div>

                <!-- Кнопка удаления -->
                <div class="row justify-end">
                  <q-btn
                    flat
                    label="Удалить"
                    color="negative"
                    icon="fa-solid fa-trash"
                    @click="openConfirmDeleteModal(coworkerItem.id)"
                  />
                </div>
              </div>
            </li>
          </ul>
        </q-card-section>
      </q-tab-panel>
    </q-tab-panels>
  </q-card>
</q-dialog>

watch(isCoworkersModalOpen, async (newVal) => {
  if (newVal) {
    // Загружаем данные для активной вкладки сразу при открытии
    if (activeCoworkersTab.value === 'recommendation') {
      await fetchCollaboratorList();
    } else {
      await fetchCoworkers();
    }
  }
});

// И в функции открытия модалки
const openCoworkersModal = async (tab = 'recommendation') => {
  activeCoworkersTab.value = tab;
  isCoworkersModalOpen.value = true;
  
  // Загружаем данные немедленно
  if (tab === 'recommendation') {
    await fetchCollaboratorList();
  } else {
    await fetchCoworkers();
  }
};

<q-tab-panels 
  v-model="activeCoworkersTab" 
  animated
  keep-alive
  class="q-pa-none"
>
