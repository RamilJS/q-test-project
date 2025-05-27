const sendDelegationInfo = async () => {
  try {
    const selectedTaskIds = selectedTasks.value.map(task => task.id);

    const requestBody = {
      action: "eval_action",
      remote_action_id: "7490881144209959956",
      wvars: [
        { name: "_object_id", value: "" },
        { name: "_secid", value: wtSecId },
        { name: "user_id", value: "" },
        { name: "adaptation_id", value: "" },
        { name: "role", value: "collaborator" },
        { name: "action_name", value: "" },
        { name: "collaborator_id", value: selectedCollaborator.value?.person?.person_id || "" },
        { name: "mentor_id", value: currentMentor.value || "" },
        { name: "mentor_tasks", value: JSON.stringify(selectedTaskIds) },
      ],
    };

    const formData = new FormData();
    formData.append("action", JSON.stringify(requestBody));
    const queryString = new URLSearchParams({ secid: wtSecId }).toString();

    await axios.post(`${BACKEND_POST_URL}${queryString}`, formData, {
      headers: { "Content-Type": "multipart/form-data" },
    });

    showToast("Делегирование выполнено");
  } catch (error) {
    console.error("Ошибка при делегировании", error);
    showToast("Ошибка при делегировании задач");
  }

  // Сброс и обновление
  await closeDelegationModal();
  selectedTasks.value = [];
  await fetchUsersData();
};
