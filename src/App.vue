<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

const collaboratorSearch = ref('');
const collaboratorListData = ref([]);

watch(collaboratorSearch, async () => {
  await fetchCollaboratorList();
});

const fetchCollaboratorList = async () => {
  try {
    const params = {
      collection_code: "uni_catalog_list",
      secid: wtSecId,
      limit: 100,
      start: 0,
      sort: JSON.stringify([{ property: "d1", direction: "ASC" }]),
      parameters: `data_mode=collaborator_list;search_str=${collaboratorSearch.value};secid=${wtSecId}`,
    };

    const response = await axios.post(
      BACKEND_URL,
      new URLSearchParams(params).toString()
    );

    collaboratorListData.value = response.data.results || [];
  } catch (error) {
    console.error("Ошибка при загрузке сотрудников:", error);
  }
};

<input
  v-model="collaboratorSearch"
  placeholder="Поиск сотрудника..."
  class="border px-4 py-2 rounded w-full"
/>
