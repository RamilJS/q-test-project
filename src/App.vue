<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<!-- Пагинация -->
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


