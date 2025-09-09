<template>
  <!-- Кнопка завершения адаптации -->
  <q-btn
    v-if="process.title === 'Подведение итогов'"
    color="primary"
    label="Завершить адаптацию"
    size="sm"
    @click="showConfirm = true"
  />

  <!-- Диалог подтверждения -->
  <q-dialog v-model="showConfirm" persistent>
    <q-card>
      <q-card-section class="text-h6">
        Вы уверены?
      </q-card-section>

      <q-card-actions align="right">
        <q-btn flat label="НЕТ" color="negative" v-close-popup />
        <q-btn flat label="ДА" color="primary" @click="onConfirm" />
      </q-card-actions>
    </q-card>
  </q-dialog>
</template>

<script setup>
import { ref } from 'vue'

const showConfirm = ref(false)

function onConfirm () {
  showConfirm.value = false
  completeAdaptation()
}

// Твой метод (он уже есть в коде)
function completeAdaptation () {
  // логика завершения адаптации
}
</script>
