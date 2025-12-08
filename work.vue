<q-input
  dense
  filled
  v-model="newMeetingEmailDisplay"
  placeholder="Введите email через ;"
  class="col"
/>

<q-btn
  icon="add"
  dense
  flat
  round
  @click="collaboratorDialogOpen = true"
/>

<script setup>
import { ref } from "vue";
import axios from "axios";

// --- Форма встречи ---
const newMeetingModalOpen = ref(false);
const meetingModalId = ref(null);

const newMeetingTopic = ref("");
const newMeetingLocation = ref("");
const newMeetingDateStart = ref("");
const newMeetingDateEnd = ref("");
const newMeetingTimeStart = ref("");
const newMeetingTimeEnd = ref("");

// --- Сотрудники ---
const selectedCollaborator = ref(null);
const collaboratorSearch = ref("");
const collaboratorDialogOpen = ref(false);
const collaboratorListData = ref([]);
const collaboratorPage = ref(1);

// --- Emails ---
const newMeetingEmail = ref([]);

// computed для отображения в input
const newMeetingEmailDisplay = computed({
  get() {
    return newMeetingEmail.value.join("; ");
  },
  set(value) {
    newMeetingEmail.value = value
      .split(";")
      .map(e => e.trim())
      .filter(e => e);
  },
});

// --- Функция сброса формы ---
const resetMeetingForm = () => {
  newMeetingTopic.value = "";
  newMeetingLocation.value = "";
  newMeetingDateStart.value = "";
  newMeetingDateEnd.value = "";
  newMeetingTimeStart.value = "";
  newMeetingTimeEnd.value = "";
  newMeetingEmail.value = [];
  selectedCollaborator.value = null;
};

// --- Отправка новой встречи ---
const postNewMeeting = async (adaptationId) => {
  try {
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
        // email отправляем через join(";")
        {
          name: "event_recipient_email",
          value: newMeetingEmail.value.join(";"),
        },
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

  // Очистка формы
  newMeetingModalOpen.value = false;
  meetingModalId.value = null;
  resetMeetingForm();
  collaboratorSearch.value = "";
  await fetchUsersData();
};
</script>


//тут
const postNewMeeting = async (adaptationId) => {
  try {
    const collaborator = selectedCollaborator.value || {};

    // гарантируем массив
    let emailsArray = [];
    if (Array.isArray(newMeetingEmail.value)) {
      emailsArray = newMeetingEmail.value;
    } else if (typeof newMeetingEmail.value === "string") {
      emailsArray = newMeetingEmail.value
        .split(";")
        .map(e => e.trim())
        .filter(e => e);
    }

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
        { name: "event_recipient_email", value: emailsArray.join(";") },
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

  newMeetingModalOpen.value = false;
  meetingModalId.value = null;
  resetMeetingForm();
  collaboratorSearch.value = "";
  await fetchUsersData();
};
