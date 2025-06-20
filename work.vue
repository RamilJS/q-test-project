// ... остальной код без изменений ...

// Измененная функция отправки комментария
const postCommentState = async (taskId) => {
  if (!newCommentText.value.trim()) return;

  try {
    const requestBody = {
      action: "eval_action",
      remote_action_id: "",
      wvars: [
        { name: "_object_id", value: "" },
        { name: "_secid", value: wtSecId },
        { name: "user_id", value: "" },
        { name: "adaptation_id", value: String(adaptationId) },
        { name: "role", value: "collaborator" },
        { name: "action_name", value: "add_comment_task" },
        { name: "task_id", value: taskId },
        { name: "task_status", value: "" },
        { name: "task_comment", value: String(newCommentText.value) },
        { name: "comment_text", value: "" },
      ],
    };

    const formData = new FormData();
    formData.append("action", JSON.stringify(requestBody));
    const queryString = new URLSearchParams({
      secid: wtSecId,
    }).toString();

    await axios.post(`${BACKEND_POST_URL}${queryString}`, formData, {
      headers: { "Content-Type": "multipart/form-data" },
    });
  } catch (error) {
    console.error("Ошибка при добавлении комментария", error);
    showToast("Ошибка при добавлении комментария");
    throw error; // Пробрасываем ошибку для обработки выше
  }
};

// Переработанная функция-обработчик отправки
const submitComment = async (taskId) => {
  if (!newCommentText.value.trim()) {
    showToast("Комментарий не может быть пустым");
    return;
  }

  try {
    // 1. Сначала отправляем комментарий
    await postCommentState(taskId);
    
    // 2. Только после успешной отправки обновляем данные
    await fetchCabinetData();
    await fetchCommentListData();
    
  } catch (error) {
    console.error("Ошибка в процессе отправки", error);
  } finally {
    // 3. Очищаем состояние независимо от результата
    newCommentText.value = "";
    isCommentModalOpen.value = false;
    newCommentModalId.value = null;
  }
};

// ... остальной код без изменений ...
