<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<q-item-label>Выберите задачу</q-item-label>

<div v-for="task in currentTaskList" :key="task.id" class="q-mb-sm">
  <q-radio
    :label="task.name"
    :value="task.id"
    v-model="optionGroup"
    color="primary"
  />
</div>
