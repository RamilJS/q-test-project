<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<ul>
  <li
    v-for="employee in underGoingAdaptation"
    :key="employee.person.person_id"
    class="adaptation-item"
  >
    <q-expansion-item
      expand-separator
      switch-toggle-side
      :label="employee.person.fullname"
      :model-value="openInfoMap[employee.person.person_id] || false"
      @update:model-value="val => toggleExpansion(employee.person.person_id, val)"
    >
      <q-card>
        <q-card-section>
          <!-- Содержимое для каждого сотрудника -->
        </q-card-section>
      </q-card>
    </q-expansion-item>
  </li>
</ul>

const toggleExpansion = (personId, value) => {
  openInfoMap.value[personId] = value;
  sessionStorage.setItem("openInfoMap", JSON.stringify(openInfoMap.value));
};

onMounted(() => {
  // Восстанавливаем состояние раскрытия
  const savedMap = sessionStorage.getItem("openInfoMap");
  if (savedMap) {
    try {
      openInfoMap.value = JSON.parse(savedMap);
    } catch (e) {
      console.error("Ошибка при разборе openInfoMap из sessionStorage:", e);
    }
  }

  // Загружаем данные, если необходимо
  fetchMaterialsData();
  fetchUsersData();
});

const openInfoMap = ref({}); // Храним состояния раскрытия для каждого элемента

// Это будет выглядеть так:
// openInfoMap.value = {
//   'employeeId1': true,
//   'employeeId2': false,
//   // и т.д.
// };

