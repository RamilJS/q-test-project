const postCommentState = (taskId) => {
  if (!newCommentText.value.trim()) return;

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
  const queryString = new URLSearchParams({ secid: wtSecId }).toString();

  axios
    .post(`${BACKEND_POST_URL}${queryString}`, formData, {
      headers: { "Content-Type": "multipart/form-data" },
    })
    .then(() => delay(300)) // 👈 ждём перед обновлением
    .then(() => {
      return Promise.all([fetchCabinetData(), fetchCommentListData()]);
    })
    .then(() => {
      newCommentText.value = "";
      isCommentModalOpen.value = false;
      newCommentModalId.value = null;
    })
    .catch((error) => {
      console.error("Ошибка при добавлении комментария", error);
      showToast("Ошибка при добавлении комментария");
    });
};
