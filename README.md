# Quasar App (q-test-project)

 quasar app for test

## Install the dependencies
```bash
yarn
# or
npm install
```

### Start the app in development mode (hot-code reloading, error reporting, etc.)
```bash
quasar dev
```


### Lint the files
```bash
yarn lint
# or
npm run lint
```



### Build the app for production
```bash
quasar build
```

### Customize the configuration
See [Configuring quasar.config.js](https://v2.quasar.dev/quasar-cli-vite/quasar-config-js).
<div v-if="activeTab === 'pending'" class="stages-list-wrapper">
  <!-- Список Невыполненных задач внутри процесса обучения -->
  <ul class="stages-list">
    <li v-for="(task, taskIndex) in pendingTasks" :key="taskIndex" class="stages-item">
      {{ task.description }}
    </li>
  </ul>
</div>

<script>
import { ref, computed } from "vue";

export default {
  setup() {
    const processes = ref([
      {
        title: "Знакомство с командой",
        state: "Выполнено",
        myEducation: "",
        tasks: [
          {
            description: "Я принял участие во встрече с руководителем и помощником по адаптации",
            duration: "1-2 рабочий день",
            durationExpire: true,
            completed: true,
            canChange: true,
            materials: [],
          },
          {
            description: "Я познакомился с командой",
            duration: "1-2 рабочий день",
            durationExpire: false,
            completed: false,
            canChange: true,
            materials: [
              {
                materialName: "Коллеги, рекомендованные для знакомства",
                materialLink: "https://als-wt/hme"
              }
            ],
          },
        ],
      },
    ]);

    
    const activeTab = ref("all");

    // Фильтр невыполненных задач
    const pendingTasks = computed(() => {
      return processes.value.flatMap((process) =>
        process.tasks.filter((task) => !task.completed)
      );
    });

    return {
      processes,
      activeTab,
      pendingTasks,
    };
  },
};
</script>

<q-toggle
  v-if="task.canChange"
  v-model="task.completed"
  :false-value="false"
  :true-value="true"
  :label="task.completed ? 'Выполнено' : 'Не выполнено'"
  color="green"
  @update:model-value="() => showToast('Задача сохранена')"
  style="width: fit-content"
  :style="task.completed ? {} : { '--q-primary': 'red' }"
/>

