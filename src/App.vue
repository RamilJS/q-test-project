<template>
  <div id="q-app">
    <router-view />
  </div>
</template

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
