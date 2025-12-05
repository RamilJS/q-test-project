<script setup>
import { ref } from "vue"
import axios from "axios"
import { useRoute } from "vue-router"

const BACKEND_POST_URL = "ТВОЙ_URL_ДЛЯ_POST?";
const wtSecId = "ТВОЙ_SECID?";

const route = useRoute()

// ---- ПЕРЕМЕННЫЕ БЕЗ ПОСРЕДНИКОВ ----
const selectedTaskId = ref(null)
const newTaskActualDate = ref("")
const selectedDateMonth = ref("")
const isDateModalOpen = ref(false)

// ---- ОТКРЫТИЕ МОДАЛКИ ----
const openDateModal = (monthTitle, taskId, actualDate) => {
  selectedDateMonth.value = monthTitle
  selectedTaskId.value = taskId
  newTaskActualDate.value = actualDate || ""   // сразу дата
  isDateModalOpen.value = true
}

// ---- СБРОС ----
const clearDateModal = () => {
  selectedTaskId.value = null
  newTaskActualDate.value = ""
  selectedDateMonth.value = ""
}

// ---- ПРЯМОЕ СОХРАНЕНИЕ В БЕК ----
const saveDateTask = async () => {
  try {
    const adaptationId = route.params.id;

    const requestBody = {
      action: "eval_action",
      remote_action_id: "7490881144209959956",
      wvars: [
        { name: "_object_id", value: "" },
        { name: "_secid", value: wtSecId },
        { name: "user_id", value: "" },
        { name: "adaptation_id", value: String(adaptationId) },
        { name: "role", value: "collaborator" },
        { name: "action_name", value: "save_date" },

        // --- ВАЖНО: ПРЯМОЕ ПЕРЕДАЧА ID + ДАТА ---
        { name: "task_id", value: String(selectedTaskId.value) },
        { name: "actual_date", value: String(newTaskActualDate.value) },
      ],
    };

    const formData = new FormData();
    formData.append("action", JSON.stringify(requestBody));

    const queryString = new URLSearchParams({ secid: wtSecId }).toString();

    const response = await axios.post(
      `${BACKEND_POST_URL}${queryString}`,
      formData,
      { headers: { "Content-Type": "multipart/form-data" } }
    );

    console.log("Ответ:", response.data);

  } catch (error) {
    console.error("Ошибка при обновлении даты задачи", error);
  }

  isDateModalOpen.value = false;
  clearDateModal();
};
</script>


<template>
  <div>
    <!-- Пример кнопки для открытия модалки -->
    <!-- month.title, task.id, task.actualDueDate -->
    <button @click="openDateModal('Январь', 123, '2025-02-03')">
      Открыть модалку
    </button>

    <!-- МОДАЛКА -->
    <div v-if="isDateModalOpen" class="modal">
      <h3>Выбрана задача: {{ selectedTaskId }}</h3>
      <h4>Месяц: {{ selectedDateMonth }}</h4>

      <input
        type="date"
        v-model="newTaskActualDate"
      />

      <button @click="saveDateTask">Сохранить</button>
      <button @click="isDateModalOpen = false">Отмена</button>
    </div>
  </div>
</template>


<style scoped>
.modal {
  padding: 20px;
  background: #fff;
  border: 1px solid #ccc;
}
</style>
