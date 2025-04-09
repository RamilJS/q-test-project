<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<template>
  <div class="employee-adaptation-container text-info-item column">
    <div class="text-info-wrapper column">
      <p class="text-info-title text-grey q-mb-sm">
        Помощник по адаптации
      </p>
    </div>

    <div class="text-info-wrapper" style="margin-bottom: 8px">
      <p class="text-info-value text-secondary">
        {{ selectedEmployee.mentor?.person_fullname || 'Не назначен' }}
      </p>
    </div>

    <div class="text-info-wrapper">
      <q-btn
        color="primary"
        text-color="white"
        label="Назначить помощника"
        size="sm"
        padding="sm"
        @click="openAppointAssistantModal"
      />
    </div>

    <!-- Модалка выбора помощника -->
    <q-dialog v-model="isAppointAssistantOpen">
      <q-card style="min-width: 650px; max-width: 800px">
        <q-card-section class="row" style="justify-content: space-between; padding-bottom: 0;">
          <div class="text-h6">Назначение помощника</div>
          <q-btn flat label="Закрыть" color="primary" v-close-popup />
        </q-card-section>

        <q-card-section>
          <!-- Имя сотрудника -->
          <div class="text-info-item column" style="margin-bottom: 15px">
            <div class="text-info-wrapper">
              <p class="text-info-title text-grey q-mb-sm">Сотрудник</p>
            </div>
            <div class="text-info-wrapper">
              <p class="text-info-value text-black">
                {{ selectedEmployee.person.person_fullname }}
              </p>
            </div>
          </div>

          <!-- Список выбора помощника -->
          <q-select
            outlined
            v-model="selectedEmployee.mentor"
            :options="selectedEmployee.mentor.change_list"
            option-label="person_fullname"
            label="Выбор помощника"
            color="grey"
            bg-color="white"
          />

          <!-- Данные выбранного помощника -->
          <div
            v-if="selectedEmployee.mentor"
            class="row"
            style="justify-content: space-between; margin-top: 15px;"
          >
            <div class="column">
              <div class="text-bold">ФИО</div>
              <q-input
                filled
                v-model="selectedEmployee.mentor.person_fullname"
                :dense="dense"
              />
            </div>

            <div class="column">
              <div class="text-bold">Должность</div>
              <div>{{ selectedEmployee.mentor.position || '—' }}</div>
            </div>

            <div class="column">
              <div class="text-bold">Структурное подразделение</div>
              <div>{{ selectedEmployee.mentor.department || '—' }}</div>
            </div>
          </div>

          <!-- Делегирование задач -->
          <div class="delegate-container" style="margin-top: 15px">
            <q-option-group
              :options="delegatedTasks"
              type="radio"
              v-model="optionGroup"
            />
          </div>
        </q-card-section>

        <q-card-actions align="right">
          <q-btn label="Отмена" color="white" text-color="primary" />
          <q-btn label="Выполнить" color="primary" @click="openDelegationError" />
        </q-card-actions>
      </q-card>
    </q-dialog>

    <!-- Модалка предупреждения -->
    <q-dialog v-model="isAppointAssistantClosing">
      <q-card style="min-width: 650px; max-width: 800px">
        <q-card-section class="row" style="justify-content: space-between; padding-bottom: 0;">
          <div class="text-h6 text-red">
            Обратите внимание, что при делегировании<br />
            остались задачи, которые не могут быть делегированы!
          </div>
          <q-btn flat label="Закрыть" color="primary" v-close-popup />
        </q-card-section>

        <q-card-section>
          <p class="text-bold">Список задач, которые не могут быть делегированы</p>
          <ul style="list-style-type: none; padding: 0;">
            <li>Направьте команде письмо-представление на нового сотрудника</li>
            <li>Представьте нового сотрудника команде</li>
            <li>Включите сотрудника в регулярные встречи</li>
            <li>Включите сотрудника в групповые чаты в мессенджерах</li>
          </ul>
        </q-card-section>
      </q-card>
    </q-dialog>
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

