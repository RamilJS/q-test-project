<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<!-- Поле поиска коллег -->
<div class="employee-search-container">
  <span class="text-bold">Рекомендации коллег</span>

  <q-input
    v-model="collaboratorSearch"
    @update:model-value="fetchCollaboratorSearch"
    dense
    outlined
    placeholder="Поиск"
    class="bg-white col"
    style="margin-top: 8px"
  />
</div>

<!-- Список рекомендованных коллег -->
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
      <div>{{ collaborator.department_name }}</div>
    </div>
  </div>
</div>


