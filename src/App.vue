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
</script>

.header-content {
  display: flex;
  align-items: center;
  width: 100%;
}

.adaptation-process-state {
  display: flex;
  align-items: center;
  margin-left: auto; /* Смещаем вправо */
}

.adaptation-state-value {
  margin-right: 8px; /* Отступ между текстом и иконкой */
}
<q-expansion-item
  expand-separator
  switch-toggle-side
  label="Первый месяц работы"
  class="expansion-item-wrapper"
>
  <template v-slot:header>
    <div class="header-content">
      <span>Первый месяц работы</span>
      <div class="adaptation-process-state row">
        <p class="adaptation-state-value">Выполнено</p>
        <q-icon size="md" name="fa-solid fa-square-check" color="green" />
      </div>
    </div>
  </template>

  <q-card>
    <q-card-section class="main-process-container column">
      <ul class="table-list">
        <li class="table-row-item column">
          ОЛтклтпу уптушкпщшуп упштущктпшщукп...
        </li>
      </ul>
    </q-card-section>
  </q-card>
</q-expansion-item>
