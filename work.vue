<template>
  <q-dialog v-model="isCoworkersModalOpen">
    <q-card style="min-width: 800px; padding: 16px;">

      <!-- Вкладки -->
      <div class="row q-gutter-sm">
        <button
          class="tab-button"
          :class="{ active: activeCoworkersTab === 'recommendation' }"
          @click="activeCoworkersTab = 'recommendation'"
        >
          Рекомендовать коллег
        </button>
        <button
          class="tab-button"
          :class="{ active: activeCoworkersTab === 'list' }"
          @click="activeCoworkersTab = 'list'"
        >
          Список коллег
        </button>
      </div>

      <q-separator class="q-my-md" />

      <!-- Контент вкладок -->
      <div v-if="activeCoworkersTab === 'recommendation'">
        <!-- Сюда помещаем форму рекомендации -->
        <CoworkerRecommendationForm />
      </div>

      <div v-else-if="activeCoworkersTab === 'list'">
        <!-- Сюда помещаем список коллег -->
        <CoworkerList />
      </div>
    </q-card>
  </q-dialog>
</template>

<script setup>
import { ref, watch } from 'vue'

// Модальное окно
const isCoworkersModalOpen = ref(false)
const activeCoworkersTab = ref('recommendation')

// Пример функции открытия
const openCoworkersModal = () => {
  activeCoworkersTab.value = 'recommendation'
  isCoworkersModalOpen.value = true
}
</script>

<style scoped>
.tab-button {
  padding: 8px 16px;
  background: #f0f0f0;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.tab-button.active {
  background: #00AAFF;
  color: white;
  font-weight: bold;
}
</style>
