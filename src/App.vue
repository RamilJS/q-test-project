<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<ul>
  <li class="adaptation-item">
    <q-expansion-item
      expand-separator
      switch-toggle-side
      :label="'Проходят адаптацию (' + underGoingAdaptation.length + ')'"
      :model-value="isExpansionOpen1"
      @update:model-value="val => toggleExpansion('isExpansionOpen1', val)"
    >
      <q-card>
        <q-card-section>
          <!-- Содержимое первого элемента -->
        </q-card-section>
      </q-card>
    </q-expansion-item>
  </li>

  <li class="adaptation-item">
    <q-expansion-item
      expand-separator
      switch-toggle-side
      :label="'Готовятся к приему (' + preparingForReception.length + ')'"
      :model-value="isExpansionOpen2"
      @update:model-value="val => toggleExpansion('isExpansionOpen2', val)"
    >
      <q-card>
        <q-card-section>
          <!-- Содержимое второго элемента -->
        </q-card-section>
      </q-card>
    </q-expansion-item>
  </li>
  <!-- Другие элементы -->
</ul>

import { ref, onMounted } from 'vue';

export default {
  setup() {
    // Состояние раскрытия каждого элемента
    const isExpansionOpen1 = ref(false);
    const isExpansionOpen2 = ref(false);
    // Другие состояния для остальных элементов

    // Функция для сохранения состояния в sessionStorage
    const toggleExpansion = (state, value) => {
      state.value = value;
      sessionStorage.setItem(state, JSON.stringify(value));
    };

    // Восстановление состояния из sessionStorage
    onMounted(() => {
      const savedState1 = sessionStorage.getItem('isExpansionOpen1');
      const savedState2 = sessionStorage.getItem('isExpansionOpen2');
      if (savedState1 !== null) {
        isExpansionOpen1.value = JSON.parse(savedState1);
      }
      if (savedState2 !== null) {
        isExpansionOpen2.value = JSON.parse(savedState2);
      }
    });

    return {
      isExpansionOpen1,
      isExpansionOpen2,
      toggleExpansion
    };
  }
};
