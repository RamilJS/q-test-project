<template>
  <!-- Замените q-input type="date" на: -->
  <q-input
    v-model="newTaskDate"
    label="Срок выполнения"
    outlined
    required
    style="max-width: 200px"
  >
    <template v-slot:append>
      <q-icon name="event" class="cursor-pointer">
        <q-popup-proxy cover transition-show="scale" transition-hide="scale">
          <q-date 
            v-model="newTaskDate" 
            mask="DD.MM.YYYY"
            :locale="ruLocale"
          />
        </q-popup-proxy>
      </q-icon>
    </template>
  </q-input>
</template>

<script>
import { ref } from 'vue'

export default {
  setup() {
    const newTaskDate = ref("")
    
    // Русская локаль для календаря
    const ruLocale = {
      days: 'Вс_Пн_Вт_Ср_Чт_Пт_Сб'.split('_'),
      daysShort: 'Вс_Пн_Вт_Ср_Чт_Пт_Сб'.split('_'),
      months: 'Январь_Февраль_Март_Апрель_Май_Июнь_Июль_Август_Сентябрь_Октябрь_Ноябрь_Декабрь'.split('_'),
      monthsShort: 'Янв_Фев_Мар_Апр_Май_Июн_Июл_Авг_Сен_Окт_Ноя_Дек'.split('_'),
      firstDayOfWeek: 1 // Понедельник
    }
    
    return { newTaskDate, ruLocale }
  }
}
</script>
