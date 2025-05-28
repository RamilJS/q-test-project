const fetchUsersData = async () => {
  try {
    const params = {
      collection_code: "",
      parameters: "",
    };

    const response = await axios.post(
      BACKEND_URL,
      new URLSearchParams(params).toString()
    );

    usersData.value = response.data.results;

    underGoingAdaptation.value = response.data.results.filter(
      (elem) => elem.stage === "Проходят адаптацию"
    );

    preparingForReception.value = response.data.results.filter(
      (elem) => elem.stage === "Готовятся к приему"
    );

    completedAdaptation.value = response.data.results.filter(
      (elem) => elem.stage === "Завершили адаптацию"
    );

    wasFiredOnAdaptation.value = response.data.results.filter(
      (elem) => elem.stage === "Были уволены в процессе адаптации"
    );

    // Сюда будем сохранять все disabled задачи
    disabledDelegationTasks.value = [];

    // Рекурсивная функция для сбора disabled задач
    const collectDisabledTasks = (tasks) => {
      for (const task of tasks) {
        if (task.disabled) {
          disabledDelegationTasks.value.push(task);
        }
        if (task.tasks && task.tasks.length > 0) {
          collectDisabledTasks(task.tasks);
        }
      }
    };

    // Проходим по каждому пользователю и собираем disabled задачи
    for (const user of response.data.results) {
      const taskList = user.mentor?.task_list || [];
      collectDisabledTasks(taskList);
    }

  } catch (error) {
    console.error("Ошибка загрузки кабинета руководителя", error);
  }
};
