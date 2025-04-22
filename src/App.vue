<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<q-card-section class="q-pt-none">
  <div class="row q-col-gutter-md">
    <div class="col">
      <q-input
        v-model="collaboratorSearch"
        placeholder="Поиск сотрудника..."
        outlined
        dense
        @keyup.enter="fetchCollaboratorList"
      />
    </div>
    <div>
      <q-btn color="primary" label="Найти" @click="fetchCollaboratorList" />
    </div>
  </div>

  <q-list bordered class="q-mt-md">
    <q-item
      v-for="item in collaboratorListData"
      :key="item.person_id"
      clickable
      @click="selectCollaborator(item)"
    >
      <q-item-section>{{ item.fullname }}</q-item-section>
    </q-item>
  </q-list>
</q-card-section>
