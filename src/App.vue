<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<script>
import { defineComponent } from 'vue'

export default defineComponent({
  name: 'App'
})

<template>
  <li class="table-row-item date-pick">
    <q-input v-model="formattedDate" filled label="Выберите дату" readonly>
      <template v-slot:append>
        <q-icon name="event" class="cursor-pointer" @click="showDatePicker = true" />
      </template>

      <q-popup-proxy v-model="showDatePicker" transition-show="scale" transition-hide="scale">
        <q-date v-model="selectedDate" mask="YYYY-MM-DD" @update:model-value="updateFormattedDate" />
      </q-popup-proxy>
    </q-input>
  </li>
</template>

<script setup>
import { ref, computed } from 'vue'
import { date } from 'quasar'

const selectedDate = ref('2025-03-23') // Начальная дата в формате YYYY-MM-DD
const showDatePicker = ref(false)

// Форматируем дату для отображения в инпуте (DD.MM.YYYY)
const formattedDate = computed(() => {
  return selectedDate.value ? date.formatDate(selectedDate.value, 'DD.MM.YYYY') : ''
})

// Функция для обновления формата даты после выбора в календаре
const updateFormattedDate = (val) => {
  selectedDate.value = val
  showDatePicker.value = false
}
</script>

