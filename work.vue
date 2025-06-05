<template>
  <div class="main-container row justify-between">
    <div class="content-container">
      <div class="new-card">
        <q-card class="my-card">
          <q-card-section class="q-pt-none">
            <div class="panel-container q-mt-sm">
              <div class="main-panel column">
                <div class="main-tasks-container">
                  <div class="tasks-name">
                    <div class="row">
                      <h3 class="person-name">
                        Задачи на испытательный срок для: {{ personFullname }}
                      </h3>
                      <q-btn
                        flat
                        text-color="primary"
                        :label="allTaskExpanded ? 'Свернуть все' : 'Развернуть все'"
                        size="sm"
                        padding="xs"
                        style="margin-bottom: auto; margin-top: auto; width: fit-content;"
                        @click="toggleProcessList"
                      />
                    </div>
                  </div>

                  <div class="tasks-table column">
                    <div class="tasks-table-header column">
                      <ul class="table-header table-list">
                        <li class="table-header-item">Задачи</li>
                        <li class="table-header-item">
                          Статус
                          <br />
                          <span class="text-grey" style="font-size: 12px">
                            заполняется <br />
                            сотрудником
                          </span>
                        </li>
                        <li class="table-header-item">
                          Подтверждение статуса
                          <br />
                          <span class="text-grey" style="font-size: 12px">
                            заполняется руководителем, наставником
                          </span>
                        </li>
                        <li class="table-header-item">Срок выполнения</li>
                        <li class="table-header-item">Дата фактического выполнения</li>
                        <li class="table-header-item">Комментарий</li>
                      </ul>

                      <ul class="month-list">
                        <li v-for="month in cabinetData" :key="month.id" class="month-item">
                          <div class="month-wrapper row" style="flex-wrap: nowrap">
                            <q-expansion-item
                              v-model="expandedTaskProcesses[month.id]"
                              expand-separator
                              switch-toggle-side
                              :label="month.title"
                              class="expansion-item-wrapper"
                              @update:model-value="value => saveExpansionState(month.id, value)"
                            >
                              <template v-slot:header>
                                <div class="header-content">
                                  <span class="header-content-month">{{ month.title }}</span>
                                </div>
                              </template>

                              <q-card>
                                <q-card-section class="main-process-container column">
                                  <ul class="month-tasks-list">
                                    <li
                                      v-for="task in month.tasks"
                                      :key="task.id"
                                      class="month-task-item"
                                    >
                                      <ul class="table-list">
                                        <li>{{ task.title }}</li>
                                        <!-- Остальные поля -->
                                      </ul>
                                    </li>
                                  </ul>
                                </q-card-section>
                              </q-card>
                            </q-expansion-item>
                          </div>
                        </li>
                      </ul>
                    </div>
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

<script>
import { ref, computed, watch, nextTick, onMounted, onUnmounted } from 'vue'
import axios from 'axios'

export default {
  setup() {
    const personFullname = ref('Иван Иванов') // Пример
    const cabinetData = ref([]) // Здесь будут приходить данные с сервера
    const expandedTaskProcesses = ref({})

    // ✅ РЕАКТИВНО вычисляется по состоянию всех expansion-элементов
    const allTaskExpanded = computed(() =>
      cabinetData.value.length > 0 &&
      cabinetData.value.every((month) => expandedTaskProcesses.value[month.id])
    )

    // ✅ Восстановление состояния после загрузки данных
    watch(cabinetData, async (newData) => {
      if (!newData || newData.length === 0) return
      await nextTick()

      newData.forEach((month) => {
        const stored = sessionStorage.getItem(`expandedMonth_${month.id}`)
        const isExpanded = stored ? JSON.parse(stored) : false
        expandedTaskProcesses.value[month.id] = isExpanded
      })
    }, { immediate: true })

    const saveExpansionState = (monthId, value) => {
      expandedTaskProcesses.value[monthId] = value
      sessionStorage.setItem(`expandedMonth_${monthId}`, JSON.stringify(value))
    }

    const toggleProcessList = () => {
      const newState = !allTaskExpanded.value
      cabinetData.value.forEach((month) => {
        expandedTaskProcesses.value[month.id] = newState
        sessionStorage.setItem(`expandedMonth_${month.id}`, JSON.stringify(newState))
      })
    }

    // ✅ Пример запроса на сервер (можешь заменить своим API)
    const loadCabinetData = async () => {
      const { data } = await axios.get('/api/cabinet-data') // Заменить на актуальный маршрут
      cabinetData.value = data
    }

    // Автоматическая загрузка при монтировании
    onMounted(() => {
      loadCabinetData()
    })

    // Стилевые мутации (если тебе нужно сохранить)
    let observer = null
    onMounted(() => {
      observer = new MutationObserver(async () => {
        await nextTick()
        // стилизация по DOM, если нужно
      })
      observer.observe(document.querySelector('.content-container'), {
        childList: true,
        subtree: true,
        attributes: true,
      })
    })

    onUnmounted(() => {
      if (observer) observer.disconnect()
    })

    return {
      personFullname,
      cabinetData,
      expandedTaskProcesses,
      allTaskExpanded,
      toggleProcessList,
      saveExpansionState,
    }
  }
}
</script>

<style scoped>
/* Добавь свои стили, если нужно */
</style>
