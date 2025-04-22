<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

const collaboratorSearch = ref("");
const selectedCoworkerId = ref(null);
const collaboratorPage = ref(1); // Для постраничной загрузки

const columnsCoworkers = ref([
  { name: 'radio', label: '', align: 'center', field: 'id', sortable: false },
  { name: 'fullname', label: 'ФИО', align: 'left', field: 'fullname' },
  { name: 'position_name', label: 'Должность', align: 'left', field: 'position_name' },
  { name: 'subdivision_name', label: 'Подразделение', align: 'left', field: 'subdivision_name' },
  { name: 'email', label: 'Email', align: 'left', field: 'email' },
  { name: 'org_name', label: 'Организация', align: 'left', field: 'org_name' },
]);

<q-input
  v-model="collaboratorSearch"
  @update:model-value="fetchCollaboratorSearch"
  dense
  outlined
  placeholder="Поиск"
  class="bg-white col"
  style="margin-top: 8px"
/>

<q-table
  :rows="collaboratorListData"
  :columns="columnsCoworkers"
  row-key="id"
  flat
  bordered
>
  <template v-slot:body="props">
    <tr>
      <td class="text-center">
        <q-radio
          v-model="selectedCoworkerId"
          :val="props.row.id"
          color="primary"
        />
      </td>
      <td>{{ props.row.fullname }}</td>
      <td>{{ props.row.position_name }}</td>
      <td>{{ props.row.subdivision_name }}</td>
      <td>{{ props.row.email }}</td>
      <td>{{ props.row.org_name }}</td>
    </tr>
  </template>
</q-table>

const openCoworkersModal = async () => {
  isCoworkersModalOpen.value = true;
  await fetchCollaboratorList();
};

const fetchCollaboratorSearch = async () => {
  try {
    const params = {
      collection_code: "vtbl_adaptation_manager_2025",
      secid: wtSecId,
      limit: 10,
      page: 0,
      start: 0,
      parameters: `data_mode=collaborator_list;search_str=${collaboratorSearch.value};secid=${wtSecId}`,
    };

    const response = await axios.post(
      BACKEND_URL,
      new URLSearchParams(params).toString()
    );

    collaboratorListData.value = response.data.results;
  } catch (error) {
    console.error("Ошибка при поиске сотрудников:", error);
  }
};

