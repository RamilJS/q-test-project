<template>
  <div class="fullscreen bg-blue text-white text-center q-pa-md flex flex-center">
    <div>
      <div style="font-size: 30vh">
        404
      </div>

      <div class="text-h2" style="opacity:.4">
        Oops. Nothing here...
      </div>

      <q-btn
        class="q-mt-xl"
        color="white"
        text-color="blue"
        unelevated
        to="/"
        label="Go Home"
        no-caps
      />
    </div>
  </div>
</template>

<script>
async postTaskState(newValue, taskId) {
  try {
    // Создаем объект с параметрами запроса
    const params = new URLSearchParams();
    params.append("action", "eval_action");
    params.append("remote_action_id", "0x657A265843D9142D");

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

    // Кодируем `wvars` как JSON-строку и добавляем в параметры
    params.append("wvars", JSON.stringify(wvars));

    // Формируем Query String с secID
    const queryString = new URLSearchParams({ secID: wtSecId }).toString();

    // Отправляем POST-запрос с URL-кодированными данными
    const response = await axios.post(`${BACKEND_URL}?${queryString}`, params.toString(), {
      headers: {
        "Content-Type": "application/x-www-form-urlencoded"
      }
    });

    this.cabinetData = response.data.results;
    console.log("Данные с сервера:", this.cabinetData);
  } catch (error) {
    console.error("Ошибка при обновлении состояния задачи", error);
    this.showToast("Ошибка при обновлении задачи");
  }
}


</script>
