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
  <ul class="table-list">
    <li class="table-row-item column">
      РВииггаи тщкуткш кп33 55 кптшк кпшткп кштпктш
    </li>

    <li class="table-row-item column">
      <q-toggle :false-value="false" :true-value="true" label="Не выполнено" color="green" unchecked-color="red" />
      <q-toggle :false-value="false" :true-value="true" label="Выполнено" color="green" unchecked-color="red" />
    </li>

    <li class="table-row-item column">
      <q-toggle :false-value="false" :true-value="true" label="Не выполнено" color="green" disable />
      <q-toggle :false-value="false" :true-value="true" label="Не выполнено" color="green" disable />
    </li>

    <li class="table-row-item">
      12.04.2025 <br />
      17.04.2025
    </li>

    <!-- Date Picker -->
    <li class="table-row-item date-pick">
      <q-input v-model="selectedDate" filled readonly label="Выберите дату">
        <template v-slot:append>
          <q-icon name="event" class="cursor-pointer" @click="showDatePicker = true" />
        </template>

        <q-popup-proxy v-model="showDatePicker" transition-show="scale" transition-hide="scale">
          <q-date v-model="selectedDate" mask="DD.MM.YYYY" @update:model-value="showDatePicker = false" />
        </q-popup-proxy>
      </q-input>
    </li>

    <li class="table-row-item">
      цель выполнена тысытв ымвтгым мывгштгвытм мвмт
    </li>
  </ul>
</template>

<script setup>
import { ref } from 'vue'

const selectedDate = ref('23.03.2025') // Начальная дата
const showDatePicker = ref(false)
</script>

