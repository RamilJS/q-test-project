<template>
  <q-dialog v-model="isOpen" persistent>
    <q-card class="q-pa-md" style="max-width: 600px; width: 100%">
      <q-card-section>
        <div class="text-h6">Задачи, которые не могут быть делегированы</div>
      </q-card-section>

      <q-separator />

      <q-card-section class="scroll q-py-sm" style="max-height: 300px">
        <q-list bordered separator>
          <q-item v-for="task in disabledDelegationTasks" :key="task.id">
            <q-item-section>{{ task.name }}</q-item-section>
          </q-item>
        </q-list>
      </q-card-section>

      <q-separator />

      <q-card-actions align="right">
        <q-btn flat label="Понятно" color="primary" @click="closeDialog" />
      </q-card-actions>
    </q-card>
  </q-dialog>
</template>

<script setup>
import { defineProps, defineEmits } from 'vue';

const props = defineProps({
  disabledDelegationTasks: {
    type: Array,
    required: true
  },
  isOpen: {
    type: Boolean,
    required: true
  }
});

const emit = defineEmits(['update:isOpen']);

function closeDialog() {
  emit('update:isOpen', false);
}
</script>
