<template>
  <div class="main-container row justify-between">
    <div class="content-container">
      <!-- Хлебные крошки -->

      <div
        class="breadcrumbs-container q-mt-sm q-gutter-sm row justify-between"
      >
        <q-breadcrumbs class="q-mt-lg g-ml-lg text-primary">
          <q-breadcrumbs-el
            class="breadcrumb-item"
            label="Учебный центр"
            @click="reloadBreadcrumb($event)"
          />
          <q-breadcrumbs-el :label="category.title" />
        </q-breadcrumbs>
      </div>

      <!-- Карточка курса -->

      <div class="new-card">
        <q-card class="my-card">
          <!--Шапка карточки курса -->

          <q-img
            class="card-header-bg-image"
            src="../images/card-header-panel.png"
          >
            <!--Плашка состояния прохождения курса -->

            <!-- Курс в процессе v-if="category.state === 'in-progress'" -->

            <div
              v-if="category.state === 'in-progress'"
              class="course-state in-progress"
            >
              <q-img src="../icons/in-progress-state.svg" />
            </div>

            <!-- Курс назначен v-else-if="category.state === 'assigned'"-->

            <div
              v-else-if="category.state === 'assigned'"
              class="course-state assigned"
            >
              <q-img src="../icons/assigned.svg" />
            </div>

            <!-- Курс не назначен v-else -->

            <div v-else class="course-state not-assigned">
              <q-img src="../icons/not-assigned.svg" />
            </div>

            <div class="header-content-container row">
              <!--Блок с текстом в шапке карточки -->

              <div class="header-content-text text-white">
                <!-- Блок с заголовком курса в шапке -->

                <p
                  class="header-content-title text-3 text-weight-regular text-uppercase text-white text-weight-bold"
                >
                  {{ category.title }}
                </p>

                <p class="header-content-description ellipsis-3-lines">
                  {{ category.cardDescription }}
                </p>
              </div>

              <!-- Картинка с сотрудниками в шапке курса -->
            </div>
          </q-img>

          <q-card-section class="q-pt-none">
            <div class="container row">
              <div class="column items-start q-pl-sm q-ml-xl q-mt-sm">
                <!-- Основная цель курса -->

                <div
                  v-if="category.target !== ''"
                  class="purpose-container q-mt-lg row no-wrap"
                >
                  <div class="purpose-trophy">
                    <img
                      src="../icons/purpose.svg"
                      alt="Иконка цели обучения"
                    />
                  </div>

                  <div class="main-purpose q-ml-md">
                    <p
                      class="main-purpose-title text-secondary text-h6 text-black q-mb-sm text-bold"
                      style="font-size: 1.5em"
                    >
                      Основная цель
                    </p>

                    <p
                      class="main-purpose-description text-black q-mb-md ellipsis-2-lines"
                    >
                      {{ category.target }}
                    </p>
                  </div>
                </div>

                <!-- Основные темы курса -->

                <div
                  v-if="mainThemes[0] !== ''"
                  class="main-themes row justify-start no-wrap q-mt-md"
                >
                  <div class="themes-icon q-mt-xs">
                    <img
                      class="courses-themes-icon"
                      src="../icons/checklist.svg"
                      alt="Иконка тем опроса"
                    />
                  </div>

                  <div class="themes-container q-ml-md">
                    <p
                      class="main-themes-title text-secondary text-h6 text-black text-bold q-mb-xs"
                      style="font-size: 1.5em"
                    >
                      Основные темы опроса
                    </p>

                    <ul class="courses-themes-list">
                      <li
                        v-for="(course, index) in mainThemes"
                        :key="index"
                        class="list-item"
                      >
                        <div class="themes-arrow-items row">
                          <div class="q-ml-xs">
                            <p
                              class="main-themes-description text-black q-mb-xs"
                            >
                              {{ course }}
                            </p>
                          </div>
                        </div>
                      </li>
                    </ul>
                  </div>
                </div>

                <!-- Блок с информацией о курсе, время, дата -->

                <div class="row justify-between q-mt-md">
                  <div
                    v-if="category.competency !== ''"
                    class="small-container competency column"
                  >
                    <div class="competency-content">
                      <p class="main-purpose-title text-black q-mb-sm">
                        Развиваемая компетенция
                      </p>
                    </div>

                    <div class="competency">
                      <p class="competency-title text-grey">
                        {{ category.competency }}
                      </p>
                    </div>
                  </div>

                  <div
                    v-if="category.time !== ''"
                    class="small-container time column"
                  >
                    <div class="time-content">
                      <p class="main-purpose-title text-black q-mb-sm">
                        Время прохождения
                      </p>
                    </div>

                    <div class="time">
                      <p class="competency-title text-grey">
                        {{ category.time }} минут
                      </p>
                    </div>
                  </div>

                  <div
                    v-if="category.dateStart !== ''"
                    class="small-container date-start column"
                  >
                    <div class="date-start-content">
                      <p class="main-date-start-title text-black q-mb-sm">
                        Опрос назначен
                      </p>
                    </div>

                    <div class="date-start-item">
                      <p class="date-start-title text-grey">
                        {{ category.dateStart }}
                      </p>
                    </div>
                  </div>

                  <div
                    v-if="category.dateFinish !== ''"
                    class="small-container date-finish column"
                  >
                    <div class="date-finish-content">
                      <p class="main-date-finish-title text-black q-mb-sm">
                        Пройти опрос до
                      </p>
                    </div>

                    <div class="date-finish-item">
                      <p class="date-finish-title text-grey">
                        {{ category.dateFinish }}
                      </p>
                    </div>
                  </div>
                </div>

                <!-- Блок с кнопками управления курсом-->

                <div
                  v-if="courseButtons.length !== 0"
                  class="course-buttons q-mt-sm row justify-between"
                >
                  <q-btn
                    v-if="category.buttons[0].command === 'start'"
                    class="start-button text-white"
                    label="Начать прохождение"
                    size="15px"
                    glossy
                    icon="fa-solid fa-play"
                    @click="onStartCourse(category.buttons[0].url)"
                  />

                  <q-btn
                    v-else-if="category.buttons[0].command === 'continue'"
                    class="start-button text-white"
                    label="Продолжить прохождение"
                    size="15px"
                    glossy
                    icon="fa-solid fa-circle-notch"
                    @click="onContinueCourse(category.buttons[0].url)"
                  />

                  <q-btn
                    v-if="category.buttons[1]"
                    class="cancel-button text-white"
                    label="Отменить назначение"
                    size="15px"
                    glossy
                    icon="fa-solid fa-ban"
                    @click="cancelCourse"
                  />

                  <!-- Сообщение об отмене -->

                  <div v-if="cancelMessage" class="cancel-message">
                    {{ cancelMessage }}
                  </div>
                </div>

                <!-- Количество попыток прохождения курса -->

                <div
                  v-if="category.courseAttempts !== ''"
                  class="attempts-count q-mt-lg q-mb-md column"
                >
                  <p class="attempts-title text-grey q-mb-xs">
                    История прохождения
                  </p>

                  <a :href="category.attemptsLink" class="attempts-link">
                    {{ category.courseAttempts }} попытки
                  </a>
                </div>

                <!-- Ссылка для оставления отзыва о курсе -->

                <div class="review q-mt-lg q-mb-md">
                  <div
                    class="review-link-container"
                    v-if="category.reviewLink !== '#'"
                  >
                    <a :href="category.reviewLink" class="review-link">
                      Оставить отзыв об опросе
                    </a>
                  </div>
                </div>
              </div>
            </div>
          </q-card-section>
        </q-card>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import { useRoute, useRouter } from "vue-router";
import axios from "axios";

const ORIGIN_URL = window.location.origin;
const BACKEND_URL = `${ORIGIN_URL}/pp/Ext5/extjs_json_collection_data.html`;
const CANCEL_MESSAGE_TIMEOUT = 3000;

const route = useRoute();
const router = useRouter();

// Состояние
const category = ref({});
const mainThemes = ref([]);
const courseButtons = ref([]);
const cancelMessage = ref(""); // Для отображения сообщения об отмене

// Загрузка данных опроса
async function fetchCourseData() {
  try {
    const params = {
      collection_code: "vtbl_quasar_survey_data",
      parameters: "survey_id=" + route.params.id,
    };
    const { data } = await axios.post(
      BACKEND_URL,
      new URLSearchParams(params).toString(),
    );
    const courseData = data.results[0]; // Данные опроса
    category.value = courseData;
    mainThemes.value = courseData.courses; // Основные темы
    courseButtons.value = courseData.buttons; // Кнопки управления
  } catch (error) {
    console.error("Ошибка загрузки опроса", error);
  }
}

// Временный показ сообщения об отмене
function showCancelMessage(message) {
  cancelMessage.value = message;
  setTimeout(() => {
    cancelMessage.value = ""; // Очистить сообщение через некоторое время
  }, CANCEL_MESSAGE_TIMEOUT);
}

// Обработчики кнопок
function onStartCourse(url) {
  window.location.href = url;
}

function onContinueCourse(url) {
  window.location.href = `${ORIGIN_URL}/${url}`;
}

async function cancelCourse() {
  try {
    const cancelUrl = `${category.value.buttons[1].url}`;
    const { data } = await axios.post(cancelUrl); // Проверка ответа сервера
    if (data.is_done === 1) {
      // Успешная отмена — обновить данные опроса
      fetchCourseData();
    } else {
      // Отобразить сообщение об ошибке от сервера
      showCancelMessage(data.message);
    }
  } catch (error) {
    console.error("Ошибка при отмене опроса:", error);
    showCancelMessage("Произошла ошибка при отмене опроса.");
  }
}

// Навигация по хлебным крошкам
function reloadBreadcrumb(evt) {
  evt.preventDefault();
  const path = "/home";
  if (route.path === path) {
    window.location.reload();
    return;
  }
  // Осуществляем переход на новый маршрут
  router
    .push(path)
    .then(() => {
      window.location.reload();
    })
    .catch((err) => {
      console.error("Error navigation:", err);
    });
}

// Скрытие лишних блоков главной страницы
function hiddenExtraHomePage() {
  document.querySelectorAll(".zone-main").forEach((el) => {
    el.style.display = "none";
  });
}

onMounted(() => {
  fetchCourseData();
  hiddenExtraHomePage();
});
</script>

<style scoped>
.card-header-bg-image {
  height: 360px;
  background:
    radial-gradient(
      68% 100% at 100% 0%,
      rgb(119, 41, 255),
      rgba(116, 42, 245, 0)
    ),
    radial-gradient(
      100% 98% at 100% 100%,
      rgb(4, 165, 236),
      rgba(41, 102, 227, 0)
    ),
    linear-gradient(0deg, rgb(51, 90, 228), rgb(16, 25, 131));
}
.breadcrumb-item {
  cursor: pointer;
  color: #4176EA;
}
.small-container {
  margin-right: 30px;
}
.small-container:last-child {
  margin-right: 0;
}
.breadcrumb-item:hover {
  color: #355bbb;
  text-decoration: none;
}
.course-state {
  position: relative;
  left: 10px;
  top: 5px;
  width: 109px;
  height: 31px;
  background: none;
}
.header-content-container {
  position: absolute;
  max-width: 70% !important;
  top: 15%;
  background: none;
  padding-bottom: 0px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.header-content-text {
  background: none;
  padding-left: 16px;
}
.header-content-title {
  font-size: 32px;
  margin-bottom: 6px;
  line-height: 1.25;
}
.header-content-description {
  color: #aae3ff;
  font-family: Roboto;
  font-size: 16px;
  margin-bottom: 0;
}
.main-purpose-description {
  max-width: 700px;
}
.course-buttons {
  width: 640px;
}

.attempts-link {
  color: #00aaff;
}
.review-link {
  color: #00aaff;
}

.container {
  width: 100%;
  min-width: 610px;
}
.courses-themes-list {
  padding-left: 20px;
}
.list-item {
  list-style-type: disc;
}

.content-container {
  flex: 1;
  background-color: unset !important;
}
.main-container {
  height: 100%;
  line-height: 1.4;
}
.start-button {
  background: #00aaff !important;
  border-radius: 5px;
  width: 20em;
}
.cancel-button {
  background: #e24f55 !important;
  border-radius: 5px;
  width: 20em;
}

.breadcrumbs-container {
  display: flex;
  justify-content: center;
  align-items: center;
  margin: 0 auto;
}

.new-card {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-top: 20px;
  margin-bottom: 40px;
}

@media (min-width: 1920px) {
  .my-card {
    width: 1200px;
  }
}

@media (min-width: 1600px) and (max-width: 1919px) {
  .my-card {
    width: 1176px;
  }
}

@media (min-width: 1373px) and (max-width: 1599px) {
  .my-card {
    width: 949px;
  }
}

@media (max-width: 1372px) {
  .my-card {
    width: 800px;
  }
  .header-content-title {
    font-size: 26px;
    margin-bottom: 6px;
    line-height: 1.25;
  }
}
</style>
