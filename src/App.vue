<template>
  <div class="q-ml-md">
    <q-checkbox
      :label="task.name"
      :model-value="isChecked"
      :indeterminate="isIndeterminate"
      @update:model-value="toggleSelection"
      color="primary"
    />
    <div v-if="task.children" class="q-ml-lg">
      <TaskCheckboxTree
        v-for="child in task.children"
        :key="child.id"
        :task="child"
        :selected="selected"
      />
    </div>
  </div>
</template>

<script setup>
import { computed } from 'vue';
import TaskCheckboxTree from './TaskCheckboxTree.vue'; // рекурсия

const props = defineProps({
  task: Object,
  selected: Array
});

const emit = defineEmits(['update:selected']);

// Все ID подзадач
const getAllTaskIds = (task) => {
  let ids = [task.id];
  if (task.children) {
    task.children.forEach(child => {
      ids = ids.concat(getAllTaskIds(child));
    });
  }
  return ids;
};

const isChecked = computed(() => {
  return getAllTaskIds(props.task).every(id => props.selected.includes(id));
});

const isIndeterminate = computed(() => {
  const allIds = getAllTaskIds(props.task);
  const selectedCount = allIds.filter(id => props.selected.includes(id)).length;
  return selectedCount > 0 && selectedCount < allIds.length;
});

const toggleSelection = (checked) => {
  const allIds = getAllTaskIds(props.task);
  let updated = [...props.selected];

  if (checked) {
    updated = Array.from(new Set([...updated, ...allIds]));
  } else {
    updated = updated.filter(id => !allIds.includes(id));
  }

  emit('update:selected', updated);
};
</script>

<!-- Контейнер для иерархического выбора задач -->
<div class="delegate-container" style="margin-top: 15px">
  <div
    v-for="task in currentTaskList"
    :key="task.id"
    class="q-mb-sm"
  >
    <TaskCheckboxTreeInline
      :task="task"
      :selected="selectedTaskIds"
      @update:selected="val => selectedTaskIds = val"
    />
  </div>
</div>

import { ref, computed, defineComponent, h } from 'vue';

const selectedTaskIds = ref([]);

// Функция рекурсивно собирает все ID из задачи и её подзадач
const getAllTaskIds = (task) => {
  let ids = [task.id];
  if (task.children) {
    task.children.forEach(child => {
      ids = ids.concat(getAllTaskIds(child));
    });
  }
  return ids;
};

// Встроенный компонент для рекурсивной отрисовки чекбоксов
const TaskCheckboxTreeInline = defineComponent({
  name: 'TaskCheckboxTreeInline',
  props: {
    task: Object,
    selected: Array
  },
  emits: ['update:selected'],
  setup(props, { emit }) {
    const allIds = computed(() => getAllTaskIds(props.task));

    const isChecked = computed(() =>
      allIds.value.every(id => props.selected.includes(id))
    );

    const isIndeterminate = computed(() => {
      const selectedCount = allIds.value.filter(id =>
        props.selected.includes(id)
      ).length;
      return selectedCount > 0 && selectedCount < allIds.value.length;
    });

    const toggleSelection = (checked) => {
      let updated = [...props.selected];
      if (checked) {
        updated = Array.from(new Set([...updated, ...allIds.value]));
      } else {
        updated = updated.filter(id => !allIds.value.includes(id));
      }
      emit('update:selected', updated);
    };

    return () =>
      h('div', [
        h('q-checkbox', {
          label: props.task.name,
          modelValue: isChecked.value,
          indeterminate: isIndeterminate.value,
          'onUpdate:modelValue': toggleSelection,
          color: 'primary',
          class: 'q-ml-sm'
        }),
        props.task.children &&
          h(
            'div',
            { class: 'q-ml-lg' },
            props.task.children.map(child =>
              h(TaskCheckboxTreeInline, {
                key: child.id,
                task: child,
                selected: props.selected,
                onUpdateSelected: (val) => emit('update:selected', val)
              })
            )
          )
      ]);
  }
});

{ name: "mentor_tasks", value: JSON.stringify(selectedTaskIds.value) },
