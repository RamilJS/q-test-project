<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<q-btn
  dense
  flat
  round
  icon="more_vert"
  @click="collaboratorDialogOpen = true"
/>

<q-dialog v-model="collaboratorDialogOpen">
  <q-card style="min-width: 400px; max-width: 600px;">
    <q-card-section>
      <div class="text-h6">Выбор сотрудника</div>
    </q-card-section>

    <q-card-section class="q-pt-none">
      <q-input
        dense
        filled
        v-model="collaboratorSearch"
        placeholder="Поиск по имени"
        @input="fetchCollaboratorList"
      />

      <q-list bordered>
        <q-item
          v-for="item in collaboratorListData"
          :key="item.person_id"
          clickable
          @click="selectCollaborator(item)"
        >
          <q-item-section>{{ item.fullname }}</q-item-section>
        </q-item>
      </q-list>

      <div class="row justify-between items-center q-mt-sm">
        <q-btn
          flat
          label="Назад"
          :disable="collaboratorPage <= 1"
          @click="collaboratorPage-- && fetchCollaboratorList()"
        />
        <q-btn
          flat
          label="Вперёд"
          @click="collaboratorPage++ && fetchCollaboratorList()"
        />
      </div>
    </q-card-section>

    <q-card-actions align="right">
      <q-btn flat label="Закрыть" v-close-popup />
    </q-card-actions>
  </q-card>
</q-dialog>

const collaboratorDialogOpen = ref(false);
const collaboratorSearch = ref('');
const collaboratorPage = ref(1);

const fetchCollaboratorList = async () => {
  try {
    const searchStr = collaboratorSearch.value || '';
    const pageSize = 10;
    const pageIndex = collaboratorPage.value - 1;

    const params = {
      collection_code: "vtbl_adaptation_manager_2025",
      secid: wtSecId,
      parameters: `data_mode=collaborator_list;paging_size=${pageSize};paging_index=${pageIndex};search_str=${searchStr};secid=${wtSecId}`,
    };

    const response = await axios.post(
      BACKEND_URL,
      new URLSearchParams(params).toString()
    );

    collaboratorListData.value = response.data.results;
  } catch (error) {
    console.error("Ошибка загрузки списка сотрудников", error);
  }
};

const selectCollaborator = (item) => {
  newMeetingEmail.value = item.email || '';
  collaboratorDialogOpen.value = false;
};

return {
  // ...
  collaboratorDialogOpen,
  collaboratorSearch,
  collaboratorPage,
  selectCollaborator,
};
const fetchCollaboratorList = async () => {
  try {
    const pageSize = 10;
    const page = collaboratorPage.value;
    const start = (page - 1) * pageSize;

    const params = {
      collection_code: "uni_catalog_list", // или твой конкретный code
      secid: wtSecId,
      limit: pageSize,
      page: page,
      start: start,
      sort: JSON.stringify([{ property: "d1", direction: "ASC" }]),
      parameters: `data_mode=collaborator_list;search_str=${collaboratorSearch.value || ''};secid=${wtSecId}`,
    };

    const response = await axios.post(
      BACKEND_URL,
      new URLSearchParams(params).toString()
    );

    collaboratorListData.value = response.data.results;
  } catch (error) {
    console.error("Ошибка при загрузке сотрудников:", error);
  }
};

