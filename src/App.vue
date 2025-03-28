<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<template>
  <div class="q-pa-md">
    <q-table
      :rows="optionsCoworkers"
      :columns="columnsCoworkers"
      row-key="id"
      flat
      bordered
    >
      <template v-slot:body="props">
        <tr>
          <td>
            <q-checkbox
              v-model="props.row.selected"
              :label="props.row.name"
              color="primary"
            />
          </td>
          <td>{{ props.row.position }}</td>
          <td>{{ props.row.subdivision }}</td>
        </tr>
      </template>
    </q-table>
  </div>
</template>

<script>
import { ref } from 'vue';

export default {
  setup() {
    const columnsCoworkers = ref([
      { name: 'name', label: 'ФИО', align: 'left', field: 'name' },
      { name: 'position', label: 'Должность', align: 'left', field: 'position' },
      { name: 'subdivision', label: 'Подразделение', align: 'left', field: 'subdivision' },
    ]);

    const optionsCoworkers = ref([
      { id: 1, name: 'Иванов Алексей Игоревич', position: 'Управляющий отделением', subdivision: 'Название подразделения', selected: false },
      { id: 2, name: 'Иванов Андрей Алексеевич', position: 'Техник по планированию', subdivision: 'Название подразделения', selected: false },
      { id: 3, name: 'Иванова Анна Ивановна', position: 'Начальник отдела по защите', subdivision: 'Название подразделения', selected: false },
      { id: 4, name: 'Иванова Дарья Владимировна', position: 'Начальник отдела кадров', subdivision: 'Название подразделения', selected: false },
      { id: 12, name: 'Иванов Андрей Алексеевич', position: 'Техник по планированию', subdivision: 'Название подразделения', selected: false },
      { id: 13, name: 'Иванова Анна Ивановна', position: 'Начальник отдела по защите', subdivision: 'Название подразделения', selected: false },
      { id: 22, name: 'Иванов Андрей Алексеевич', position: 'Техник по планированию', subdivision: 'Название подразделения', selected: false },
      { id: 33, name: 'Иванова Анна Ивановна', position: 'Начальник отдела по защите', subdivision: 'Название подразделения', selected: false },
      { id: 42, name: 'Иванов Андрей Алексеевич', position: 'Техник по планированию', subdivision: 'Название подразделения', selected: false },
      { id: 43, name: 'Иванова Анна Ивановна', position: 'Начальник отдела по защите', subdivision: 'Название подразделения', selected: false },
    ]);

    return { columnsCoworkers, optionsCoworkers };
  }
};
</script>
