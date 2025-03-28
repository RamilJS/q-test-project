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

<script>
export default {
  data() {
    return {
      columns: [
        { name: 'fio', label: 'ФИО', align: 'left', field: 'fio' },
        { name: 'position', label: 'Должность', align: 'left', field: 'position' },
        { name: 'subdivision', label: 'Подразделение', align: 'left', field: 'subdivision' },
      ],
      optionsCoworkers: [
        { id: 1, fio: 'Иванов Алексей Игоревич', position: 'Управляющий отделением', subdivision: 'Название подразделения', selected: false },
        { id: 2, fio: 'Иванов Андрей Алексеевич', position: 'Техник по планированию', subdivision: 'Название подразделения', selected: false },
        { id: 3, fio: 'Иванова Анна Ивановна', position: 'Начальник отдела по защите', subdivision: 'Название подразделения', selected: false },
        { id: 4, fio: 'Иванова Дарья Владимировна', position: 'Начальник отдела кадров', subdivision: 'Название подразделения', selected: false }
      ]
    };
  }
};
</script>

