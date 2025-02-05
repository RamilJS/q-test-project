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
.side-panel {
  position: fixed; /* Фиксируем боковую панель */
  top: 0; /* Прижимаем к верхнему краю */
  left: 0; /* Прижимаем к левому краю (если панель слева) */
  width: 250px; /* Укажите нужную ширину */
  height: 100vh; /* Растягиваем на всю высоту экрана */
  background: rgb(241, 241, 241);
  padding: 10px;
  border-radius: 5px;
  overflow-y: auto; /* Позволяет прокручивать содержимое внутри панели */
}
