<template>
  <!-- Объединенное модальное окно -->
  <q-dialog v-model="isCoworkersModalOpen">
    <q-card style="min-width: 800px;">
      <!-- Заголовок модального окна с вкладками -->
      <q-card-section class="row" style="justify-content: space-between; padding-bottom: 0;">
        <q-tabs
          v-model="activeTab"
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

      <q-card-section>
        <!-- Содержимое вкладки формы рекомендации -->
        <q-tab-panel name="recommendation" v-show="activeTab === 'recommendation'">
          <!-- Информация о сотруднике -->
          <div class="text-info-item column" style="margin-bottom: 15px;">
            <div class="text-info-wrapper">
              <p class="text-info-title text-grey q-mb-sm">Сотрудник</p>
            </div>
            <div class="text-info-wrapper">
              <p class="text-info-value text-black">{{ personFullname }}</p>
            </div>
          </div>

          <!-- Поиск коллег -->
          <div class="employee-search-container">
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

          <!-- Список коллег -->
          <div class="q-pa-md" style="padding-left: 0; padding-right: 0;">
            <div
              v-for="collaborator in collaboratorListData"
              :key="collaborator.id"
              class="q-pa-sm q-mb-sm q-border q-rounded bg-white"
              style="display: flex; align-items: flex-start;"
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

          <!-- Раздел комментариев -->
          <div class="coworker-comment" style="margin-top: 20px;">
            <span class="text-grey">Комментарий</span>
            <q-input
              v-model="newComment.text"
              label="Введите описание"
              filled
              clearable
              type="textarea"
              autogrow
              :maxlength="1000"
              style="margin: 15px 0;"
            />
            <span class="text-grey">
              Опишите, по какому вопросу будет полезен каждый рекомендованный коллега
            </span>
          </div>

          <!-- Кнопки действий -->
          <q-card-actions align="right" style="margin-bottom: 8px;">
            <q-btn
              label="Отмена"
              color="white"
              text-color="primary"
              style="margin-right: 10px"
              @click="isCoworkersModalOpen = false"
            />
            <q-btn
              label="Порекомендовать коллег"
              color="primary"
              @click="saveCoworkersRecommendation()"
            />
          </q-card-actions>
        </q-tab-panel>

        <!-- Содержимое вкладки списка рекомендованных -->
        <q-tab-panel name="list" v-show="activeTab === 'list'">
          <ul class="coworkers-list">
            <li
              v-for="coworkerItem in coworkersListData"
              :key="coworkerItem.id"
              class="coworker-item row"
            >
              <!-- Аватар коллеги -->
              <div class="coworker-avatar" style="margin-right: 10px;">
                <q-avatar
                  v-if="coworkerItem.person_icon_url"
                  size="100px"
                  class="q-mr-sm"
                >
                  <img
                    :src="coworkerItem.person_icon_url"
                    style="object-fit: cover;"
                  />
                </q-avatar>
                <q-avatar v-else size="100px" class="q-mr-sm">
                  <img src="../images/person-default.png" />
                </q-avatar>
              </div>

              <!-- Информация о коллеге -->
              <div class="coworker-info column text-grey">
                <div class="coworker-info-name text-black">
                  <span>{{ coworkerItem.fullname }}</span>
                </div>
                <div class="coworker-info-position">
                  <span>{{ coworkerItem.position_name }}</span>
                </div>
                <div class="coworker-info-subdivision">
                  <span>{{ coworkerItem.subdivision_name }}</span>
                </div>
                <div class="coworker-info-email">
                  <span>{{ coworkerItem.email }}</span>
                </div>
                <div class="coworker-info-phone">
                  <span v-if="coworkerItem.phone">{{ coworkerItem.phone }}</span>
                </div>
                <div class="coworker-info-comments text-black row" style="margin-top: 10px;">
                  <span style="padding-bottom: 10px; padding-right: 10px;">
                    Комментарий:
                  </span>
                  <span style="padding: 10px; padding-top: 0; padding-left: 0;">
                    {{ coworkerItem.comment }}
                  </span>
                </div>
                <div class="row">
                  <q-btn
                    flat
                    label="Удалить"
                    color="negative"
                    @click="openConfirmDeleteModal(coworkerItem.id)"
                  />
                </div>
              </div>
            </li>
          </ul>
        </q-tab-panel>
      </q-card-section>
    </q-card>
  </q-dialog>

  <!-- Модальное окно успеха -->
  <q-dialog v-model="isCoworkersSuccessModalOpen">
    <q-card style="min-width: 500px">
      <q-card-section class="text-h6">
        {{ coworkersMessage }}
      </q-card-section>
      <q-card-actions align="right">
        <q-btn label="Закрыть" flat color="primary" @click="isCoworkersSuccessModalOpen = false" />
        <q-btn
          label="Порекомендовать ещё коллегу"
          color="primary"
          @click="recommendAnotherCoworker"
        />
      </q-card-actions>
    </q-card>
  </q-dialog>

  <!-- Модальное окно подтверждения удаления -->
  <q-dialog v-model="isConfirmDeleteModalOpen">
    <q-card style="min-width: 400px">
      <q-card-section class="text-h6">
        Вы уверены, что хотите удалить коллегу, рекомендованного для знакомства?
      </q-card-section>
      <q-card-actions align="right">
        <q-btn label="Закрыть" flat color="primary" @click="isConfirmDeleteModalOpen = false" />
        <q-btn
          label="Удалить"
          color="negative"
          @click="confirmDeleteCoworker"
        />
      </q-card-actions>
    </q-card>
  </q-dialog>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue';
import { useRoute } from 'vue-router';
import axios from 'axios';

const BACKEND_URL = '';
const BACKEND_PERSON_URL = '';

const route = useRoute();
const isCoworkersModalOpen = ref(false);
const isCoworkersSuccessModalOpen = ref(false);
const isConfirmDeleteModalOpen = ref(false); // Для модального окна подтверждения
const activeTab = ref('recommendation');
const collaboratorSearch = ref('');
const selectedCoworkerId = ref(null);
const collaboratorPage = ref(1);
const collaboratorListData = ref([]);
const coworkersListData = ref([]);
const coworkersMessage = ref('');
const newComment = ref({ text: '' });
const personFullname = ref('');
const coworkerIdToDelete = ref(null); // Для хранения ID коллеги, которого нужно удалить

const openCoworkers = async (tab = 'recommendation') => {
  isCoworkersModalOpen.value = true;
  activeTab.value = tab;
  if (tab === 'recommendation') {
    await fetchCollaboratorList();
  } else if (tab === 'list') {
    await fetchCoworkers();
  }
};

const openConfirmDeleteModal = (id) => {
  coworkerIdToDelete.value = id;
  isConfirmDeleteModalOpen.value = true;
};

const confirmDeleteCoworker = async () => {
  if (coworkerIdToDelete.value) {
    await deleteCoworker(coworkerIdToDelete.value);
    isConfirmDeleteModalOpen.value = false;
    coworkerIdToDelete.value = null;
  }
};

const fetchCollaboratorList = async () => {
  try {
    const pageSize = 5;
    const page = collaboratorPage.value;
    const start = (page - 1) * pageSize;

    const params = {
      collection_code: 'vtbl_adaptation_manager_2025',
      secid: wtSecid,
      limit: pageSize,
      page: page,
      start: start,
      parameters: `data_mode=collaborator_list;search_str=${collaboratorSearch.value};secid=${wtSecid}`,
    };

    const response = await axios.post(
      BACKEND_URL,
      new URLSearchParams(params).toString()
    );

    collaboratorListData.value = response.data.results;
  } catch (error) {
    console.error('Ошибка при загрузке коллег:', error);
  }
};

const fetchCollaboratorSearch = async () => {
  try {
    const pageSize = 5;
    const page = 1;
    const start = 0;

    const params = {
      collection_code: 'vtbl_adaptation_manager_2025',
      secid: wtSecid,
      limit: pageSize,
      page: page,
      start: start,
      parameters: `data_mode=collaborator_list;search_str=${collaboratorSearch.value};secid=${wtSecid}`,
    };

    const response = await axios.post(
      BACKEND_URL,
      new URLSearchParams(params).toString()
    );

    collaboratorListData.value = response.data.results;
  } catch (error) {
    console.error('Ошибка при поиске коллег:', error);
  }
};

const fetchCoworkers = async () => {
  try {
    const adaptationId = route.params.id;

    const params = {
      collection_code: 'vtbl_adaptation_manager_2025',
      parameters: `data_mode=meet_collaborator;adaptation_id=${adaptationId}`,
    };

    const response = await axios.post(
      BACKEND_URL,
      new URLSearchParams(params).toString()
    );

    coworkersListData.value = response.data.results;
  } catch (error) {
    console.error('Ошибка при загрузке рекомендованных коллег:', error);
  }
};

const nextCollaboratorPage = async () => {
  collaboratorPage.value += 1;
  await fetchCollaboratorList();
};

const prevCollaboratorPage = async () => {
  if (collaboratorPage.value > 1) {
    collaboratorPage.value -= 1;
    await fetchCollaboratorList();
  }
};

const saveCoworkersRecommendation = async () => {
  try {
    const adaptationId = route.params.id;

    const requestBody = {
      action: 'eval_action',
      remote_action_id: '7490881144209959956',
      wvars: [
        { name: '_object_id', value: '' },
        { name: '_secid', value: wtSecid },
        { name: 'user_id', value: '' },
        { name: 'adaptation_id', value: String(adaptationId) },
        { name: 'role', value: 'collaborator' },
        { name: 'action_name', value: 'add_meet_with_collegue' },
        { name: 'coworker_id', value: selectedCoworkerId.value },
        { name: 'coworker_comment', value: newComment.value.text },
      ],
    };

    const formData = new FormData();
    formData.append('action', JSON.stringify(requestBody));
    const queryString = new URLSearchParams({ secid: wtSecid }).toString();

    const response = await axios.post(
      `${BACKEND_PERSON_URL}${queryString}`,
      formData,
      {
        headers: { 'Content-Type': 'multipart/form-data' },
      }
    );

    coworkersMessage.value = response.data.result.message;
    isCoworkersModalOpen.value = false;
    isCoworkersSuccessModalOpen.value = true;
  } catch (error) {
    console.error('Ошибка при сохранении рекомендации:', error);
  } finally {
    newComment.value.text = '';
    selectedCoworkerId.value = null;
    collaboratorSearch.value = '';
    await fetchCollaboratorSearch();
  }
};

const recommendAnotherCoworker = async () => {
  isCoworkersSuccessModalOpen.value = false;
  await openCoworkers('recommendation');
};

const deleteCoworker = async (ID) => {
  try {
    const adaptationId = route.params.id;

    const requestBody = {
      action: 'eval_action',
      remote_action_id: '7490881144209959956',
      wvars: [
        { name: '_object_id', value: '' },
        { name: '_secid', value: wtSecid },
        { name: 'user_id', value: '' },
        { name: 'adaptation_id', value: String(adaptationId) },
        { name: 'role', value: 'collaborator' },
        { name: 'action_name', value: 'delete_meet' },
        { name: 'coworker_id', value: String(ID) },
      ],
    };

    const formData = new FormData();
    formData.append('action', JSON.stringify(requestBody));
    const queryString = new URLSearchParams({ secid: wtSecid }).toString();

    const response = await axios.post(
      `${BACKEND_PERSON_URL}${queryString}`,
      formData,
      {
        headers: { 'Content-Type': 'multipart/form-data' },
      }
    );

    await fetchCoworkers();
    showToast('Коллега успешно удален');
  } catch (error) {
    console.error('Ошибка при удалении коллеги:', error);
    showToast('Ошибка при удалении коллеги');
  }
};

// Загрузка данных для вкладки recommendation при открытии модального окна
watch(isCoworkersModalOpen, async (isOpen) => {
  if (isOpen && activeTab.value === 'recommendation') {
    await fetchCollaboratorList();
  }
});

// Отслеживание изменений вкладки для загрузки соответствующих данных
watch(activeTab, async (newTab) => {
  if (newTab === 'recommendation') {
    await fetchCollaboratorList();
  } else if (newTab === 'list') {
    await fetchCoworkers();
  }
});

onMounted(() => {
  // Загрузка начальных данных при необходимости
});
</script>

<style scoped>
/* Добавьте специфические стили здесь */
</style>
