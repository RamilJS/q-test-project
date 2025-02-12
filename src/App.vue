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

    
.month-wrapper {
  display: flex;
  align-items: center; /* Выровнять элементы по центру */
  flex-wrap: nowrap;
}

.expansion-item-wrapper {
  display: flex;
  align-items: center; /* Чтобы label и иконка были на одной линии */
}

.adaptation-process-state {
  display: flex;
  align-items: center; /* Выровнять текст и иконку по центру */
  margin-left: auto; /* Сдвинуть вправо */
}
