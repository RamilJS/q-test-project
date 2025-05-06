const selectedCollaborator = ref(null);
const selectCollaborator = (item) => {
  newMeetingEmail.value = item.email || '';
  selectedCollaborator.value = item;
  collaboratorDialogOpen.value = false;
};


const postNewMeeting = async () => {
  try {
    const adaptationId = route.params.id;
    const collaborator = selectedCollaborator.value || {};

    const requestBody = {
      action: "eval_action",
      remote_action_id: "7490881144209959956",
      wvars: [
        { name: "_object_id", value: "" },
        { name: "_secid", value: wtSecId },
        { name: "user_id", value: "" },
        { name: "adaptation_id", value: String(adaptationId) },
        { name: "role", value: "collaborator" },

        { name: "action_name", value: "create_event" },
        { name: "event_id", value: "" },
        { name: "event_date_start", value: newMeetingDateStart.value },
        { name: "event_date_finish", value: newMeetingDateEnd.value },
        { name: "event_time_start", value: newMeetingTimeStart.value },
        { name: "event_time_finish", value: newMeetingTimeEnd.value },
        { name: "event_subject_line", value: newMeetingTopic.value },
        { name: "event_place", value: newMeetingLocation.value },
        { name: "event_recipient_email", value: newMeetingEmail.value },

        // Добавлены поля из выбранного сотрудника
        { name: "event_recipient_fullname", value: collaborator.fullname || "" },
        { name: "event_recipient_id", value: collaborator.person_id || "" },
        { name: "event_recipient_org_name", value: collaborator.org_name || "" },
        { name: "event_recipient_position_name", value: collaborator.position_name || "" },
        { name: "event_recipient_subdivision_name", value: collaborator.subdivision_name || "" },
      ],
    };

    const formData = new FormData();
    formData.append("action", JSON.stringify(requestBody));
    const queryString = new URLSearchParams({ secid: wtSecId }).toString();

    await axios.post(`${BACKEND_POST_URL}${queryString}`, formData, {
      headers: { "Content-Type": "multipart/form-data" },
    });

    showToast("Встреча успешно создана");
  } catch (error) {
    console.error("Ошибка при создании новой встречи", error);
    showToast("Ошибка при создании новой встречи");
  }

  // Очистка и закрытие
  newMeetingModalOpen.value = false;
  meetingModalId.value = null;
};
