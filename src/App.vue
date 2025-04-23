<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<div class="employee-search-container">
  <span class="text-bold">Рекомендации коллег</span>
  <div class="row q-gutter-sm" style="margin-top: 8px;">
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


