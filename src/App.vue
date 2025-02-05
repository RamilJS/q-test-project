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
  position: sticky;
  top: 10px; /* Панель "прилипнет" к верхнему краю родительского контейнера */
  height: fit-content; /* Высота подстраивается под контент */
  background: rgb(241, 241, 241);
  padding: 10px;
  border-radius: 5px;
}
