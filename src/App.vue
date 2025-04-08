<template>
  <div id="q-app">
    <router-view />
  </div>
</template>


const adaptationCategories = computed(() => [
  { label: 'Проходят адаптацию', list: underGoingAdaptation.value },
  { label: 'Готовятся к приему', list: preparingForReception.value },
  { label: 'Завершили адаптацию', list: completedAdaptation.value },
  { label: 'Были уволены в процессе адаптации', list: wasFiredOnAdaptation.value },
]);

<ul class="adaptation-list column" style="list-style-type: none; padding-left: 0; margin: 0;">
  <li
    v-for="(category, index) in adaptationCategories"
    :key="index"
    class="adaptation-item"
  >
    <q-expansion-item
      expand-separator
      switch-toggle-side
      :label="category.label + ' ' + category.list.length"
      class="expansion-item-wrapper"
    >
      <q-card v-if="category.list.length">
        <q-card-section class="main-process-container column">
          <ul class="employee-adaptation-list column">
            <li
              v-for="employee in category.list"
              :key="employee.id"
              class="employee-adaptation-item column"
            >
              <!-- тут вставь свою разметку одного сотрудника (повторяющийся блок) -->
              <EmployeeCard :employee="employee" />
            </li>
          </ul>
        </q-card-section>
      </q-card>
    </q-expansion-item>
  </li>
</ul>
