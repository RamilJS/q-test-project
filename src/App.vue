<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<template>
  <div class="q-pa-md">
    <q-table
      :rows="optionsCoworkers"
      :columns="columns"
      row-key="id"
      flat
      bordered
    >
      <template v-slot:body-cell-fio="props">
        <q-checkbox
          v-model="props.row.selected"
          :label="props.row.fio"
          color="primary"
        />
      </template>
      <template v-slot:body-cell-position="props">
        {{ props.row.position }}
      </template>
      <template v-slot:body-cell-subdivision="props">
        {{ props.row.subdivision }}
      </template>
    </q-table>
  </div>
</template>
