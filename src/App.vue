<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<script>
import { defineComponent } from 'vue'

export default defineComponent({
  name: 'App'
})
  
<template>
  <div>
    <!-- Предполагается, что объект task доступен (например, передаётся через props или из store) -->
    <q-toggle
      v-if="task.canChange"
      v-model="task.completed"
      @update:model-value="handleToggle"
      :false-value="false"
      :true-value="true"
      :label="task.completed ? 'Выполнено' : 'Не выполнено'"
      color="green"
      style="width: fit-content"
    />
  </div>
</template>

<script>
import axios from "axios";

const BACKEND_URL =
  window.location.origin + `/pp/Ext5/extjs_json_collection_data.html`;

export default {
  name: "TaskToggle",
  props: {
    // Например, задача может передаваться через пропс
    task: {
      type: Object,
      required: true,
    },
  },
  data() {
    return {
      // Пример хранения данных, полученных с сервера
      cabinetData: [],
    };
  },
  methods: {
    async handleToggle(newValue) {
      // newValue содержит новое значение переключателя (true/false)
      // Вызываем метод для отправки состояния задачи на сервер
      await this.postTaskState(this.task.id);

      // После успешного запроса показываем уведомление
      this.showToast("Задача сохранена");
    },
    async postTaskState(taskId) {
      try {
        const params = {
          collection_code: "vtbl_adaptation_2025",
          parameters: "task_id=" + taskId,
        };

        // Отправляем POST-запрос на сервер
        const response = await axios.post(
          BACKEND_URL,
          new URLSearchParams(params).toString()
        );

        // Обновляем данные, если необходимо
        this.cabinetData = response.data.results;
        console.log("Данные с сервера:", this.cabinetData);
      } catch (error) {
        console.error("Ошибка при обновлении состояния задачи", error);
        this.showToast("Ошибка при обновлении задачи");
      }
    },
    showToast(message) {
      // Если вы используете Quasar, можно воспользоваться его уведомлениями:
      this.$q.notify({
        message: message,
        color: "green",
        position: "top",
      });
      // Либо используйте любую другую реализацию уведомлений
    },
    // Пример метода, который может быть вызван в mounted
    fetchCabinetData() {
      // Логика получения данных
    },
    changeStepperStyle() {
      // Логика изменения стилей (если требуется)
    },
  },
  mounted() {
    // При монтировании компонента можно загрузить данные
    this.fetchCabinetData();
    this.changeStepperStyle();
  },
};
</script>

<template>
  <li class="table-row-item date-pick">
    <q-input v-model="formattedDate" filled label="Выберите дату" readonly>
      <template v-slot:append>
        <q-icon name="event" class="cursor-pointer" @click="showDatePicker = true" />
      </template>

      <q-popup-proxy v-model="showDatePicker" transition-show="scale" transition-hide="scale">
        <q-date v-model="selectedDate" mask="YYYY-MM-DD" @update:model-value="updateFormattedDate" />
      </q-popup-proxy>
    </q-input>
  </li>
</template>

<script setup>
import { ref, computed } from 'vue'
import { date } from 'quasar'

const selectedDate = ref('2025-03-23') // Начальная дата в формате YYYY-MM-DD
const showDatePicker = ref(false)

// Форматируем дату для отображения в инпуте (DD.MM.YYYY)
const formattedDate = computed(() => {
  return selectedDate.value ? date.formatDate(selectedDate.value, 'DD.MM.YYYY') : ''
})

// Функция для обновления формата даты после выбора в календаре
const updateFormattedDate = (val) => {
  selectedDate.value = val
  showDatePicker.value = false
}
</script>

async postTaskState(newValue, taskId) {
  try {
    const requestBody = {
      action: "eval_action",
      remote_action_id: "0x657A265843D9142D",
      wvars: [
        { name: "_object_id", value: "" },
        { name: "_secid", value: "D9B4A57C66982CDE9869AA8583A8F5AD" },
        { name: "user_id", value: "" },
        { name: "adaptation_id", value: "" },
        { name: "role", value: "collaborator" },
        { name: "action_name", value: "save_task" },
        { name: "task_id", value: taskId },
        { name: "task_status", value: newValue },
        { name: "task_comment", value: "1" }
      ]
    };

    const response = await axios.post(BACKEND_URL, requestBody, {
      headers: {
        "Content-Type": "application/json",
        "secID": wtSecId // передаем secID в заголовке
      }
    });

    this.cabinetData = response.data.results;
    console.log("Данные с сервера:", this.cabinetData);
  } catch (error) {
    console.error("Ошибка при обновлении состояния задачи", error);
    this.showToast("Ошибка при обновлении задачи");
  }
}
