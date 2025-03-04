<template>
  <div id="q-app">
    <router-view />
  </div>
</template

document.addEventListener("DOMContentLoaded", function () {
    document.querySelectorAll("span").forEach(span => {
        if (span.textContent.trim() === "Выгрузить отчёт по фактическому обучению") {
            span.textContent = "Отчёт по обучению";
        }
    });
});

  
async postTaskState(newValue, taskId) {
  try {
    // Создаем объект FormData
    const formData = new FormData();
    formData.append("action", "eval_action");
    formData.append("remote_action_id", "0x657A265843D9142D");

    const wvars = [
      { name: "_object_id", value: "" },
      { name: "_secid", value: "D9B4A57C66982CDE9869AA8583A8F5AD" },
      { name: "user_id", value: "" },
      { name: "adaptation_id", value: "" },
      { name: "role", value: "collaborator" },
      { name: "action_name", value: "save_task" },
      { name: "task_id", value: taskId },
      { name: "task_status", value: newValue },
      { name: "task_comment", value: "1" }
    ];

    formData.append("wvars", JSON.stringify(wvars)); // Добавляем массив в виде строки

    // Формируем Query String с secID
    const queryString = new URLSearchParams({ secID: wtSecId }).toString();

    // Отправляем POST-запрос с FormData и Query String
    const response = await axios.post(`${BACKEND_URL}?${queryString}`, formData, {
      headers: {
        "Content-Type": "multipart/form-data" // Указываем FormData
      }
    });

    this.cabinetData = response.data.results;
    console.log("Данные с сервера:", this.cabinetData);
  } catch (error) {
    console.error("Ошибка при обновлении состояния задачи", error);
    this.showToast("Ошибка при обновлении задачи");
  }
}


<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Копирование ссылки</title>
    <script src="https://cdn.jsdelivr.net/npm/quasar@2.14.1/dist/quasar.umd.min.js"></script>
    <link href="https://cdn.jsdelivr.net/npm/quasar@2.14.1/dist/quasar.prod.css" rel="stylesheet">
    <script src="https://cdn.jsdelivr.net/npm/vue@3.2.47/dist/vue.global.prod.js"></script>
</head>
<body>

<div id="app">
    <q-btn @click="copyToClipboard" label="Скопировать ссылку" color="primary">
        <q-tooltip v-if="copied" anchor="bottom middle" self="top middle" :offset="[10, 10]">
            Ссылка успешно скопирована!
        </q-tooltip>
    </q-btn>
</div>

<script>
const { createApp, ref } = Vue;
const { Quasar } = window;

createApp({
    setup() {
        const copied = ref(false);
        const link = "https://example.com"; // Замените на свою ссылку

        const copyToClipboard = async () => {
            try {
                await navigator.clipboard.writeText(link);
                copied.value = true;
                setTimeout(() => copied.value = false, 2000); // Убираем сообщение через 2 секунды
            } catch (err) {
                console.error("Ошибка при копировании: ", err);
            }
        };

        return { copied, copyToClipboard };
    }
}).use(Quasar).mount("#app");
</script>

</body>
</html>

