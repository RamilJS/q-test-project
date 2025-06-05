<script setup>
import { ref, computed, onMounted, watch } from 'vue'

// Модальные окна, табы и уведомления
const isCoworkersModalOpen = ref(false)
const toastVisible = ref(false)
const toastMessage = ref('')
const activeTab = ref('all')

// Данные процессов
const cabinetData = ref([])

// Состояние раскрытия по ID процесса
const expandedProcesses = ref({})

// === Загрузка данных с сервера ===
async function fetchData() {
  // Пример API вызова
  // const response = await axios.get('/api/cabinet')
  // cabinetData.value = response.data

  // 🔧 Пример заглушки:
  cabinetData.value = [
    { id: 1, title: 'Адаптация маркетолога', tasks: [{ id: 101, description: 'Ознакомиться с задачами', completed: false }] },
    { id: 2, title: 'Онбординг разработчика', tasks: [{ id: 102, description: 'Установить окружение', completed: true }] }
  ]
  restoreExpansionState()
}

// === Сохранение состояния в sessionStorage ===
function saveExpansionState() {
  Object.keys(expandedProcesses.value).forEach(id => {
    sessionStorage.setItem(`expanded_process_${id}`, JSON.stringify(expandedProcesses.value[id]))
  })
}

// === Восстановление из sessionStorage при загрузке ===
function restoreExpansionState() {
  cabinetData.value.forEach(proc => {
    const saved = sessionStorage.getItem(`expanded_process_${proc.id}`)
    expandedProcesses.value[proc.id] = saved ? JSON.parse(saved) : false
  })
}

// === Обновление одного элемента (при ручном раскрытии/сворачивании) ===
function toggleProcess(id) {
  saveExpansionState()
}

// === Вычисляемое: все ли раскрыты? ===
const allExpanded = computed(() =>
  cabinetData.value.length &&
  cabinetData.value.every(proc => expandedProcesses.value[proc.id])
)

// === Кнопка: Свернуть/Развернуть всё ===
function toggleAll() {
  const newState = !allExpanded.value
  cabinetData.value.forEach(proc => {
    expandedProcesses.value[proc.id] = newState
  })
  saveExpansionState()
}

onMounted(fetchData)
</script>

<template>
  <div class="main-container row justify-between">
    <div class="content-container">
      <q-card class="my-card">
        <q-card-section>
          <!-- Кнопка "Развернуть все / Свернуть все" -->
          <div class="row justify-end q-mb-sm">
            <q-btn
              color="primary"
              :label="allExpanded ? 'Свернуть все' : 'Развернуть все'"
              @click="toggleAll"
              size="sm"
              flat
            />
          </div>

          <div class="panel-container q-mt-sm">
            <div class="main-panel column">
              <!-- Список всех процессов -->
              <ul class="adaptation-process-list column">
                <li
                  v-for="process in cabinetData"
                  :key="process.id"
                  class="adaptation-process-item row"
                >
                  <q-expansion-item
                    v-model="expandedProcesses[process.id]"
                    expand-separator
                    switch-toggle-side
                    :label="process.title"
                    class="expansion-item-wrapper"
                    @update:model-value="toggleProcess(process.id)"
                  >
                    <q-card>
                      <q-card-section class="main-process-container column">
                        <div class="process-tab-container row">
                          <p
                            class="process-tab-item"
                            :class="{ 'process-tab-current': activeTab === 'all' }"
                            @click="activeTab = 'all'"
                          >
                            Все задачи
                          </p>
                          <p
                            class="process-tab-item"
                            :class="{ 'process-tab-current': activeTab === 'pending' }"
                            @click="activeTab = 'pending'"
                          >
                            Невыполненные задачи
                          </p>
                        </div>

                        <!-- Все задачи -->
                        <div v-if="activeTab === 'all'" class="stages-list-wrapper">
                          <ul class="stages-list">
                            <li
                              v-for="task in process.tasks"
                              :key="task.id"
                              class="stages-item"
                            >
                              {{ task.description }}
                            </li>
                          </ul>
                        </div>

                        <!-- Невыполненные -->
                        <div v-if="activeTab === 'pending'" class="stages-list-wrapper">
                          <ul
                            v-if="process.tasks.some(t => !t.completed)"
                            class="stages-list"
                          >
                            <li
                              v-for="task in process.tasks.filter(t => !t.completed)"
                              :key="task.id"
                              class="stages-item"
                            >
                              {{ task.description }}
                            </li>
                          </ul>
                          <div v-else>Все задачи выполнены</div>
                        </div>
                      </q-card-section>
                    </q-card>
                  </q-expansion-item>
                </li>
              </ul>
            </div>
          </div>
        </q-card-section>
      </q-card>
    </div>
  </div>
</template>

<style scoped>
.process-tab-item {
  margin-right: 20px;
  cursor: pointer;
}
.process-tab-current {
  font-weight: bold;
  color: #027be3;
}
</style>
