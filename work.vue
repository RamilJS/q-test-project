// Это можно использовать в data() или computed
isSpecialTask(task) {
  return task.description === 'Получите от руководителя задачи на испытательный срок';
}

<!-- Контейнер для переключателей -->
<div class="toggles-container">
  <!-- Если спец. задача — только флаг "Сотрудник" и заблокированный toggle -->
  <div v-if="task.description === 'Получите от руководителя задачи на испытательный срок'" class="employee-toggle-container">
    <div class="manager-title">
      <q-icon color="green" size="8px" name="fa-solid fa-circle" />
      <span class="manager-title-item">Сотрудник</span>
    </div>

    <q-toggle
      v-model="task.completedCollaborator"
      :false-value="false"
      :true-value="true"
      :label="task.completedCollaborator ? 'Выполнено' : 'Не выполнено'"
      color="green"
      disable
    >
      <q-tooltip>Задача сохраняется сотрудником</q-tooltip>
    </q-toggle>
  </div>

  <!-- Остальные задачи -->
  <div v-else class="main-toggle-container">
    <div class="manager-title">
      <q-icon
        :color="process.title === 'Программа обучения' ? 'green' : 'red'"
        size="8px"
        name="fa-solid fa-circle"
      />
      <span class="manager-title-item">
        {{ process.title === 'Программа обучения' ? 'Сотрудник' : 'Руководитель' }}
      </span>
    </div>

    <q-toggle
      v-if="task.canChange"
      v-model="task.completedMentor"
      :false-value="false"
      :true-value="true"
      :label="task.completedMentor ? 'Выполнено' : 'Не выполнено'"
      color="green"
      @update:model-value="handleToggle(task.completedMentor, task.id, process)"
      style="width: fit-content"
      disable
    />

    <q-toggle
      v-else
      v-model="task.completedMentor"
      :false-value="false"
      :true-value="true"
      :label="task.completedMentor ? 'Выполнено' : 'Не выполнено'"
      color="green"
      unchecked-color="red"
      disable
    >
      <q-tooltip>
        {{
          isEmployeeTask(process, task)
            ? 'Данное поле заполняется сотрудником'
            : 'Задача сохраняется автоматически'
        }}
      </q-tooltip>
    </q-toggle>
  </div>
</div>

