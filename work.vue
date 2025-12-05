<template>
  <div class="main-container row justify-between">
    <div class="content-container">
      <div class="new-card">
        <q-card class="my-card">
          <q-card-section class="q-pt-none">
            <div class="panel-container q-mt-sm">
              <div class="main-panel column">

                <div class="main-tasks-container">
                  <div class="tasks-table column">

                    <ul class="month-list">
                      <li v-for="month in cabinetData" :key="month.id" class="month-item">
                        <div class="month-wrapper row" style="flex-wrap: nowrap">

                          <q-card>
                            <q-card-section class="main-process-container column">

                              <ul class="month-tasks-list">
                                <li v-for="task in month.tasks" :key="task.id" class="month-task-item">

                                  <ul class="table-list">
                                    
                                    <!-- кнопка открытия модалки -->
                                    <li class="table-row-item">
                                      <div class="column">
                                        {{ task.actualDueDate }}

                                        <q-btn
                                          class="add-task-button"
                                          label="Добавить дату"
                                          color="secondary"
                                          @click="openDateModal(month.title, task)"
                                        />
                                      </div>
                                    </li>

                                  </ul>

                                </li>
                              </ul>

                            </q-card-section>
                          </q-card>

                        </div>
                      </li>
                    </ul>

                  </div>
                </div>

              </div>
            </div>
          </q-card-section>
        </q-card>
      </div>
    </div>
  </div>

  <!-- ========================= -->
  <!-- ОДНА ОБЩАЯ МОДАЛКА ДАТЫ  -->
  <!-- ========================= -->
  <q-dialog v-model="isDateModalOpen">
    <q-card class="column" style="min-width: 500px; max-width: 600px">

      <div class="row justify-between no-wrap">
        <span style="margin-top: 20px; margin-left: 20px">
          Дата фактического выполнения задачи
        </span>

        <q-btn icon="close" flat dense round style="margin: 10px"
               @click="closeDateModal" />
      </div>

      <q-card-section>
        <div class="q-gutter-md column">

          <div class="date-title row">
            <span>Текущая задача:</span>
            <span>{{ currentTask?.task_name }}</span>
          </div>

          <q-input
            v-model="newTaskActualDate"
            label="Срок выполнения"
            outlined
            style="max-width: 200px"
          >
            <template #append>
              <q-icon name="event" class="cursor-pointer">
                <q-popup-proxy cover transition-show="scale" transition-hide="scale">
                  <q-date
                    v-model="newTaskActualDate"
                    mask="DD.MM.YYYY"
                    :locale="ruLocale"
                  />
                </q-popup-proxy>
              </q-icon>
            </template>
          </q-input>

        </div>
      </q-card-section>

      <q-card-actions align="right">
        <q-btn flat label="Отмена" color="grey" @click="closeDateModal" />
        <q-btn flat label="Сохранить" color="secondary" @click="saveDateTask" />
      </q-card-actions>

    </q-card>
  </q-dialog>
</template>

<script>
import { ref } from "vue"
import { date } from "quasar"

export default {
  setup() {

    const isDateModalOpen = ref(false)
    const selectedDateMonth = ref(null)
    const newTaskActualDate = ref("")
    const currentTask = ref(null)

    // Русская локаль
    const ruLocale = {
      days: 'Вс_Пн_Вт_Ср_Чт_Пт_Сб'.split('_'),
      daysShort: 'Вс_Пн_Вт_Ср_Чт_Пт_Сб'.split('_'),
      months: 'Январь_Февраль_Март_Апрель_Май_Июнь_Июль_Август_Сентябрь_Октябрь_Ноябрь_Декабрь'.split('_'),
      monthsShort: 'Янв_Фев_Мар_Апр_Май_Июн_Июл_Авг_Сен_Окт_Ноя_Дек'.split('_'),
      firstDayOfWeek: 1
    }

    // Открытие модалки
    const openDateModal = (monthTitle, task) => {
      selectedDateMonth.value = monthTitle
      currentTask.value = task
      newTaskActualDate.value = task.actualDueDate || ""
      isDateModalOpen.value = true
    }

    const closeDateModal = () => {
      isDateModalOpen.value = false
      clearDateModal()
    }

    const clearDateModal = () => {
      newTaskActualDate.value = ""
      selectedDateMonth.value = ""
      currentTask.value = null
    }

    // Сохранение даты (локальная логика + API если нужно)
    const saveDateTask = () => {
      if (currentTask.value) {
        currentTask.value.actualDueDate = newTaskActualDate.value
      }

      isDateModalOpen.value = false
      clearDateModal()
    }

    return {
      isDateModalOpen,
      selectedDateMonth,
      newTaskActualDate,
      currentTask,
      ruLocale,
      openDateModal,
      closeDateModal,
      saveDateTask
    }
  }
}
</script>
