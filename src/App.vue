<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<script setup>
import { ref, computed } from "vue";

const isAppointAssistantOpen = ref(false);
const isAppointAssistantClosing = ref(false);

// Модель текущего выбранного сотрудника
const selectedEmployee = ref({
  person: {
    person_fullname: "Иванов Иван Иванович"
  },
  mentor: {
    person_fullname: "Петров Петр Петрович",
    position: "Управляющий отделением",
    department: "Название подразделения",
    change_list: [
      {
        person_fullname: "Петров Петр Петрович",
        position: "Управляющий отделением",
        department: "Название подразделения"
      },
      {
        person_fullname: "Сидоров Сидр Сидорович",
        position: "Старший специалист",
        department: "Отдел ИТ"
      }
    ]
  }
});

const delegatedTasks = ref([
  { label: "Делегировать все задачи (из возможных)", value: "1" },
  { label: "Создайте заявки из АРМ", value: "2" },
  { label: "Направьте командное письмо", value: "3" }
]);

// Radio (одиночный выбор)
const optionGroup = ref(null);

const openAppointAssistantModal = () => {
  isAppointAssistantOpen.value = true;
};

const openDelegationError = () => {
  isAppointAssistantClosing.value = true;
};
</script>

