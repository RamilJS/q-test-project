<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

const employees = ref([
  {
    id: 1,
    name: "Тестов Тест Тестович",
    position: "Кредитный аналитик",
    avatar: require("../images/person-default.png"),
    assistant: null, // или { name: "Помощник Тест Иванович", position: "..." }
    hasMeetings: false,
    adaptationEnd: "30.07.2025",
    progress: {
      now_day_count: 20,
      all_day_count: 90,
      stages: [
        { name: "1 этап", percent: 100 },
        { name: "2 этап", percent: 60 },
        { name: "3 этап", percent: 0 },
      ],
      probationTasksPercent: 50,
      trainingProgramPercent: 70,
    },
  },
  // Добавим ещё сотрудников:
  {
    id: 2,
    name: "Иванов Иван Иванович",
    position: "Менеджер проектов",
    avatar: require("../images/person-default.png"),
    assistant: { name: "Смирнов Алексей", position: "Старший менеджер" },
    hasMeetings: true,
    adaptationEnd: "15.08.2025",
    progress: {
      now_day_count: 40,
      all_day_count: 90,
      stages: [
        { name: "1 этап", percent: 100 },
        { name: "2 этап", percent: 80 },
        { name: "3 этап", percent: 20 },
      ],
      probationTasksPercent: 80,
      trainingProgramPercent: 60,
    },
  },
  {
    id: 3,
    name: "Петрова Ольга Сергеевна",
    position: "HR-специалист",
    avatar: require("../images/person-default.png"),
    assistant: null,
    hasMeetings: false,
    adaptationEnd: "01.09.2025",
    progress: {
      now_day_count: 10,
      all_day_count: 60,
      stages: [
        { name: "1 этап", percent: 50 },
        { name: "2 этап", percent: 0 },
        { name: "3 этап", percent: 0 },
      ],
      probationTasksPercent: 30,
      trainingProgramPercent: 15,
    },
  },
]);

<ul class="employee-adaptation-list column">
  <li
    class="employee-adaptation-item column"
    v-for="employee in employees"
    :key="employee.id"
  >
    <div class="employee-main-info row">
      <!-- Аватар и ФИО -->
      <div class="employee-adaptation-container column">
        <router-link
          class="employee-cabinet-link column"
          :to="`/adaptation-manager/${employee.id}`"
        >
          <q-avatar size="70px" class="q-mr-sm">
            <img :src="employee.avatar" style="object-fit: cover;" />
          </q-avatar>
          <q-tooltip>Перейти на страницу сотрудника</q-tooltip>
        </router-link>
      </div>

      <div class="employee-adaptation-container column">
        <router-link
          class="employee-cabinet-link column"
          style="text-decoration: none; color: black;"
          :to="`/adaptation-manager/${employee.id}`"
        >
          <span class="employee-name text-bold">{{ employee.name }}</span>
          <span class="employee-position">{{ employee.position }}</span>
          <q-tooltip>Перейти на страницу сотрудника</q-tooltip>
        </router-link>
      </div>

      <!-- Помощник по адаптации -->
      <div class="employee-adaptation-container text-info-item column">
        <div class="text-info-wrapper">
          <p class="text-info-title text-grey q-mb-sm">Помощник по адаптации</p>
        </div>
        <div v-if="employee.assistant" class="text-info-wrapper">
          <p class="text-info-value text-secondary">{{ employee.assistant.name }}</p>
        </div>
        <div v-else class="text-info-wrapper">
          <q-btn
            color="primary"
            text-color="white"
            label="Назначить помощника"
            size="sm"
            padding="sm"
            @click="openAppointAssistantModal(employee)"
          />
        </div>
      </div>

      <!-- Назначенные встречи -->
      <div class="employee-adaptation-container text-info-item column">
        <div class="text-info-wrapper">
          <p class="text-info-title text-grey q-mb-sm">Назначенные встречи</p>
        </div>
        <div class="text-info-wrapper">
          <p v-if="employee.hasMeetings" class="text-info-value text-secondary">
            1 встреча
          </p>
          <q-btn
            v-else
            class="new-metting-link"
            color="primary"
            text-color="white"
            label="НАЗНАЧИТЬ ВСТРЕЧУ"
            size="sm"
            padding="sm"
            @click="createNewMeeting(employee)"
          />
        </div>
      </div>

      <!-- Завершение адаптации -->
      <div class="employee-adaptation-container text-info-item column">
        <div class="text-info-wrapper">
          <p class="text-info-title text-grey q-mb-sm">Завершение адаптации</p>
        </div>
        <div class="text-info-wrapper">
          <p class="text-info-value text-secondary">{{ employee.adaptationEnd }}</p>
        </div>
      </div>

      <!-- Прогресс -->
      <div class="employee-other-info column">
        <q-stepper
          v-model="step"
          ref="stepper"
          color="secondary"
          flat
          alternative-labels
        >
          <q-step
            v-for="(stage, index) in employee.progress.stages"
            :key="index"
            :name="index"
            :title="stage.name"
            icon="fa-solid fa-check"
            :done="stage.percent === 100"
            :disable="stage.percent < 100"
            color="secondary"
            class="progress-stepper-item"
          />
        </q-stepper>

        <div class="circle-container row">
          <div class="circle-item row">
            <q-circular-progress
              show-value
              reverse
              :value="employee.progress.probationTasksPercent"
              size="60px"
              :thickness="0.22"
              color="light-blue"
              track-color="grey-3"
              class="q-ma-md"
            >
              {{ employee.progress.probationTasksPercent }}%
            </q-circular-progress>
            <p class="text-secondary" style="margin: auto 0">
              Задачи на <br /> испытательный срок
            </p>
          </div>
          <div class="circle-item row">
            <q-circular-progress
              show-value
              reverse
              :value="employee.progress.trainingProgramPercent"
              size="60px"
              :thickness="0.22"
              color="light-blue"
              track-color="grey-3"
              class="q-ma-md"
            >
              {{ employee.progress.trainingProgramPercent }}%
            </q-circular-progress>
            <p class="text-secondary" style="margin: auto 0">
              Программа обучения
            </p>
          </div>
        </div>
      </div>
    </div>
  </li>
</ul>
