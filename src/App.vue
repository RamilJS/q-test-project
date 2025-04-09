<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

const currentMentor = ref(null);
const currentChangeList = ref([]);
const currentTaskList = ref([]);

const openAppointAssistantModal = (id) => {
  appointAssistantModalId.value = id;
  isAppointAssistantOpen.value = true;

  // Найти сотрудника
  const employee = usersData.value.find(
    (emp) => emp.person.person_id === id
  );

  if (employee && employee.mentor) {
    currentMentor.value = null; // сбрасываем при открытии
    currentChangeList.value = employee.mentor.change_list || [];
    currentTaskList.value = employee.mentor.task_list || [];
  }
};

<!-- Список выбора помощника -->
<q-select
  outlined
  v-model="currentMentor"
  :options="currentChangeList"
  label="Выбор помощника"
  option-label="person_fullname"
  option-value="person_id"
  emit-value
  map-options
  color="grey"
  bg-color="white"
/>

<!-- Блок информации о выбранном менторе -->
<div
  v-if="currentMentor"
  class="row"
  style="justify-content: space-between; margin-top: 15px;"
>
  <div class="column">
    <div class="text-bold">ФИО</div>
    <div>{{ currentChangeList.find(m => m.person_id === currentMentor)?.person_fullname }}</div>
  </div>

  <div class="column">
    <div class="text-bold">Должность</div>
    <div>{{ currentChangeList.find(m => m.person_id === currentMentor)?.person_position }}</div>
  </div>

  <div class="column">
    <div class="text-bold">Структурное подразделение</div>
    <div>{{ currentChangeList.find(m => m.person_id === currentMentor)?.person_subdivision }}</div>
  </div>
</div>

<!-- Контейнер для делегирования задач -->
<div class="delegate-container" style="margin-top: 15px">
  <q-option-group
    :options="currentTaskList"
    type="radio"
    v-model="optionGroup"
    option-label="name"
    option-value="id"
  />
</div>

return {
  // ...
  currentMentor,
  currentChangeList,
  currentTaskList,
  optionGroup,
  // ...
};
