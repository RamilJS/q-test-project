<!-- Все задачи -->
<div v-if="activeTab === 'all'" class="stages-list-wrapper">
  <ul class="stages-list">
    <li v-for="task in process.tasks" :key="task.id" class="stages-item">
      <!-- Срок выполнения задачи-->
      <div class="duration-wrapper">
        <!-- ... (остальной код без изменений) ... -->
      </div>

      <!--Контейнер для переключателей toggle-->
      <div class="toggles-container">
        <!--Блок-сотрудник с переключателем состояния-->
        <div 
          v-if="process.title === 'Подведение итогов' || task?.description === 'Получите от руководителя задачи на испытательный срок'"
          class="employee-toggle-container"
        >
          <!-- Изменение 1: Скрываем автора действия для конкретной задачи -->
          <div 
            v-if="task?.description !== 'Изучите необходимые документы и материалы'"
            class="manager-title"
            style="margin-left: 10px; background-color: #ddd; border-radius: 10px; width: fit-content; padding: 0 5px;"
          >
            <q-icon color="green" size="8px" name="fa-solid fa-circle" />
            <span class="manager-title-item" style="margin-left: 5px; font-size: 12px;">
              Сотрудник
            </span>
          </div>

          <q-toggle
            v-model="task.completedCollaborator"
            :false-value="false"
            :true-value="true"
            :label="task.completedCollaborator ? 'Выполнено' : 'Не выполнено'"
            color="green"
            unchecked-color="red"
            disable
          >
            <!-- Изменение 2: Меняем текст tooltip для конкретной задачи -->
            <q-tooltip>
              {{ task?.description === 'Изучите необходимые документы и материалы' 
                  ? 'Задача сохраняется автоматически' 
                  : 'Задача сохраняется сотрудником' }}
            </q-tooltip>
          </q-toggle>
        </div>

        <!--Блок-руководитель с переключателем состояния-->
        <div 
          v-if="task?.description !== 'Получите от руководителя задачи на испытательный срок'"
          class="main-toggle-container"
        >
          <!-- Изменение 3: Скрываем автора действия для конкретной задачи -->
          <div 
            v-if="task?.description !== 'Изучите необходимые документы и материалы'"
            class="manager-title"
            style="margin-left: 10px; background-color: #ddd; border-radius: 10px; width: fit-content; padding: 0 5px;"
          >
            <q-icon
              :color="process.title === 'Программа обучения' ? 'green' : 'red'"
              size="8px"
              name="fa-solid fa-circle"
            />
            <span class="manager-title-item" style="margin-left: 5px; font-size: 12px;">
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
            <!-- Изменение 4: Меняем текст tooltip для конкретной задачи -->
            <q-tooltip>
              {{ task?.description === 'Изучите необходимые документы и материалы'
                  ? 'Задача сохраняется автоматически'
                  : (isEmployeeTask(process, task) 
                    ? 'Данное поле заполняется сотрудником' 
                    : 'Задача сохраняется автоматически') }}
            </q-tooltip>
          </q-toggle>
        </div>
      </div>

      <!-- ... (остальной код без изменений) ... -->
    </li>
  </ul>
</div>

const isEmployeeTask = (process, task) => {
  if (process.title === 'Программа обучения') {
    return task?.description !== 'Изучите необходимые документы и материалы';
  }
  return false;
};
