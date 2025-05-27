{ name: "mentor_tasks", value: JSON.stringify(selectedTaskIds.value) },

<!-- Иерархический выбор задач -->
<div class="delegate-container" style="margin-top: 15px">
  <div
    v-for="task in currentTaskList"
    :key="task.id"
    class="q-mb-sm"
  >
    <div>
      <q-checkbox
        :label="task.name"
        :model-value="isTaskChecked(task)"
        :indeterminate="isTaskIndeterminate(task)"
        @update:model-value="checked => toggleTask(task, checked)"
        color="primary"
        class="q-ml-sm"
      />
    </div>

    <div v-if="task.children" class="q-ml-lg">
      <div
        v-for="subtask in task.children"
        :key="subtask.id"
        class="q-mb-xs"
      >
        <q-checkbox
          :label="subtask.name"
          :model-value="isTaskChecked(subtask)"
          :indeterminate="isTaskIndeterminate(subtask)"
          @update:model-value="checked => toggleTask(subtask, checked)"
          color="primary"
          class="q-ml-sm"
        />

        <div v-if="subtask.children" class="q-ml-lg">
          <div
            v-for="subsubtask in subtask.children"
            :key="subsubtask.id"
            class="q-mb-xs"
          >
            <q-checkbox
              :label="subsubtask.name"
              :model-value="isTaskChecked(subsubtask)"
              @update:model-value="checked => toggleTask(subsubtask, checked)"
              color="primary"
              class="q-ml-sm"
            />
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
const selectedTaskIds = ref([]);

// Получает все вложенные ID задачи
const getAllTaskIds = (task) => {
  let ids = [task.id];
  if (task.children) {
    task.children.forEach(child => {
      ids = ids.concat(getAllTaskIds(child));
    });
  }
  return ids;
};

// Проверка, выбрана ли вся задача и её потомки
const isTaskChecked = (task) => {
  const allIds = getAllTaskIds(task);
  return allIds.every(id => selectedTaskIds.value.includes(id));
};

// Проверка, выбраны ли только некоторые из потомков
const isTaskIndeterminate = (task) => {
  const allIds = getAllTaskIds(task);
  const count = allIds.filter(id => selectedTaskIds.value.includes(id)).length;
  return count > 0 && count < allIds.length;
};

// Обработчик выбора/снятия задачи и её подзадач
const toggleTask = (task, checked) => {
  const allIds = getAllTaskIds(task);
  let updated = [...selectedTaskIds.value];

  if (checked) {
    allIds.forEach(id => {
      if (!updated.includes(id)) updated.push(id);
    });
  } else {
    updated = updated.filter(id => !allIds.includes(id));
  }

  selectedTaskIds.value = updated;
};
