<script>
import { ref, onMounted, onUnmounted, computed, watch} from "vue";
import axios from "axios";
 
const BACKEND_URL =
  window.location.origin + `/pp/Ext5/extjs_json_collection_data.html`;
const BACKEND_POST_URL =
  window.location.origin + `/lpapi.html?object_id=&doc_id=&`;
 
export default {
  setup() {
    const confirmDeleteMentorDialogOpen = ref(false);
    const mentorToDeleteId = ref(null);
    const optionGroup = ref(null);
    const selectedTasks = ref([]);
    const selectedTaskIds = ref([]);
    const disabledDelegationTasks = ref([]);
    const toastVisible = ref(false);
    const toastMessage = ref("");
    const errorToastVisible = ref(false);
    const errorToastMessage = ref("");
    const usersData = ref([]);
    const materialsData = ref([]);
    const isAppointAssistantOpen = ref(false);
    const isAppointAssistantClosing = ref(false);
    const personFullname = ref("");
    const mentorFullname = ref("");
    const userEventsLentgh = ref(null);
    const userSecondStage = ref(null);
    const userThirdStage = ref(null);
    let observer;
    const isOtherInfoOpen = ref(false);
    const openInfoMap = ref({});
    const isMeetingModalOpen = ref(false);
    const newMeetingModalOpen = ref(false);
    const newMeetingEmail = ref([]);
    const newMeetingDateStart = ref("");
    const newMeetingDateEnd = ref("");
    const newMeetingTimeStart = ref("");
    const newMeetingTimeEnd = ref("");
    const newMeetingTopic = ref("Встреча по адаптации");
    const newMeetingLocation = ref("");
    const underGoingAdaptation = ref([]); // Проходят адаптацию
    const preparingForReception = ref([]); // Готовятся к приему
    const completedAdaptation = ref([]); // Завершили адаптацию
    const wasFiredOnAdaptation = ref([]); // Были уволены в процессе адаптации
    const newEmployeeMaterials = ref([]);
    const assistantMaterials = ref([]);
    const bossMaterials = ref([]);
    const bossActionsData = ref([]);
    const meetingModalId = ref(null);
    const appointAssistantModalId = ref(null);
    const currentMentor = ref(null);
    const currentChangeList = ref([]);
    const currentTaskList = ref([]);
    const collaboratorListData = ref([]);
    const collaboratorDialogOpen = ref(false);
    const collaboratorSearch = ref('');
    const collaboratorPage = ref(1);
    const sessionID = wtSecId;
    // const selectedCollaborator = ref(null);
 
    const newMeetingEmailInput = computed({
      get() {
        return newMeetingEmail.value.join("; ");
      },
      set(val) {
        newMeetingEmail.value = val
          .split(";")
          .map(e => e.trim())
          .filter(e => e.length > 0);
      }
    });
 
    // Автоматическая дата окончания — копируем дату начала
    watch(newMeetingDateStart, (newDate) => {
      newMeetingDateEnd.value = newDate;
    });
 
    // Автоматическое время окончания — +1 час к началу
    watch(newMeetingTimeStart, (newTime) => {
      if (!newTime) return;
 
      const [hours, minutes] = newTime.split(':').map(Number);
      let newHours = hours + 1;
 
      // Учитываем переход через 23:00 -> 00:00
      if (newHours >= 24) newHours -= 24;
 
      // Форматируем обратно в строку с ведущим нулем
      const formattedHours = String(newHours).padStart(2, '0');
      const formattedMinutes = String(minutes).padStart(2, '0');
      newMeetingTimeEnd.value = `${formattedHours}:${formattedMinutes}`;
    });
 
    const showToast = (message) => {
      toastMessage.value = message;
      toastVisible.value = true;
      setTimeout(() => (toastVisible.value = false), 2000);
    };
 
    const showErrorToast = (message) => {
      errorToastMessage.value = message;
      errorToastVisible.value = true;
      setTimeout(() => (errorToastVisible.value = false), 3500);
    };
 
    // Преобразует tasks → children (Quasar требует ключ children)
    function transformTasksToTree(tasks) {
      return tasks.map(task => ({
        ...task,
        children: task.tasks ? transformTasksToTree(task.tasks) : []
      }));
    }
 
    // computed дерево
    const treeData = computed(() => {
      if (!currentTaskList.value?.length) return [];
      return transformTasksToTree(currentTaskList.value);
    });
 
    const resetCollaboratorSearch = () => {
      collaboratorSearch.value = '';
      collaboratorPage.value = 1;
      // collaboratorListData.value = [];
    };
 
    const openCollaboratorDialog = async () => {
      collaboratorDialogOpen.value = true;
    };
 
    const closeCollaboratorDialog = async () => {
      collaboratorDialogOpen.value = false;
      resetCollaboratorSearch();
      await fetchCollaboratorList();
      await fetchCollaboratorSearch();
    };
 
    const selectCollaborator = async (item) => {
 
      if (item.email && !newMeetingEmail.value.includes(item.email)) {
        newMeetingEmail.value.push(item.email);
        // selectedCollaborator.value.push(item.email);
      }
      await closeCollaboratorDialog();
    };
 
    const expansionStates = ref({
      undergoing: false,
      preparing: false,
      completed: false,
      fired: false,
    });
 
    const saveExpansionState = () => {
      sessionStorage.setItem('expansionStates', JSON.stringify(expansionStates.value));
    };
 
    const loadExpansionState = () => {
      const saved = sessionStorage.getItem('expansionStates');
      if (saved) {
        expansionStates.value = JSON.parse(saved);
      }
    };
 
    const printPage = async (printURL) => {
      window.location.href = `${printURL}${wtSid}`;
    };
 
    const openDelegationError = () => {
      isAppointAssistantClosing.value = true;
    };
 
    const closeDelegationModal = async () => {
      isAppointAssistantOpen.value = false;
    };
 
    const sendDelegationInfo = async (adaptationId) => {
      try {
        //const collaborator = selectedCollaborator.value || {};
 
        const requestBody = {
          action: "eval_action",
          remote_action_id: "7510212276354488535",
          wvars: [
            { name: "_object_id", value: "" },
            { name: "_secid", value: wtSecId },
            { name: "user_id", value: "" },
            { name: "adaptation_id", value: String(adaptationId) },
            { name: "role", value: "collaborator" },
            { name: "action_name", value: "set_mentor" },
            //{ name: "collaborator_id", value: collaborator.value.person.person_id },
            { name: "mentor_id", value: currentMentor.value },
            { name: "mentor_tasks", value: JSON.stringify(selectedTaskIds.value) },
            //{ name: "mentor_tasks", value: JSON.stringify(selectedTaskIds.value) },
          ],
        };
 
        const formData = new FormData();
        formData.append("action", JSON.stringify(requestBody));
        const queryString = new URLSearchParams({ secid: wtSecId }).toString();
 
        await axios.post(`${BACKEND_POST_URL}${queryString}`, formData, {
          headers: { "Content-Type": "multipart/form-data" },
        });
 
 
      } catch (error) {
        console.error("Ошибка при делегировании", error);
        showToast("Ошибка при делегировании задач");
      }
 
      // Сброс и обновление
      await closeDelegationModal();
      selectedTaskIds.value = [];
      isAppointAssistantClosing.value = true;
      await fetchUsersData();
    };
 
    const deleteDelegationInfo = async (adaptationId) => {
      try {
 
        const requestBody = {
          action: "eval_action",
          remote_action_id: "7510212276354488535",
          wvars: [
            { name: "_object_id", value: "" },
            { name: "_secid", value: wtSecId },
            { name: "user_id", value: "" },
            { name: "adaptation_id", value: String(adaptationId) },
            { name: "role", value: "collaborator" },
            { name: "action_name", value: "set_mentor" },
            //{ name: "collaborator_id", value: collaborator.value.person.person_id },
            { name: "mentor_id", value: "delete" },
            { name: "mentor_tasks", value: "" },
          ],
        };
 
        const formData = new FormData();
        formData.append("action", JSON.stringify(requestBody));
        const queryString = new URLSearchParams({ secid: wtSecId }).toString();
 
        await axios.post(`${BACKEND_POST_URL}${queryString}`, formData, {
          headers: { "Content-Type": "multipart/form-data" },
        });
 
 
      } catch (error) {
        console.error("Ошибка при удалении помощника", error);
        showToast("Ошибка при удалении помощника");
      }
 
      // Сброс и обновление
      //await closeDelegationModal();
      selectedTaskIds.value = [];
      showToast('Помощник успешно удален');
      await fetchUsersData();
    };
 
    const openOtherInfo = (id) => {
      openInfoMap.value[id] = !openInfoMap.value[id];
    };
 
    const createNewMeeting = async (id) => {
      meetingModalId.value = id;
      newMeetingModalOpen.value = true;
      await fetchCollaboratorList();
    };
 
    const openMeetingModal = () => {
      isMeetingModalOpen.value = true;
    };
 
    const openAppointAssistantModal = (id) => {
      appointAssistantModalId.value = id;
      isAppointAssistantOpen.value = true;
 
 
      // Находим сотрудника
    const employee = usersData.value.find(
      (em) => em.person.person_id === id
    );
 
    if (employee && employee.mentor) {
      currentMentor.value = null; // сбрасываем при открытии
      currentChangeList.value = employee.mentor.change_list || [];
      currentTaskList.value = employee.mentor.task_list || [];
 
    }
  };
 
    const fetchUsersData = async () => {
      try {
        const params = {
          collection_code: "vtbl_adaptation_tutor_2025",
          parameters: "data_mode=get_adaptation_list",
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
 
    const fetchMaterialsData = async () => {
      try {
        const params = {
          collection_code: "vtbl_adaptation_tutor_2025",
          parameters: "data_mode=get_materials",
        };
        const response = await axios.post(
          BACKEND_URL,
          new URLSearchParams(params).toString()
        );
        materialsData.value = response.data.results;
        newEmployeeMaterials.value = response.data.results.filter(
          (elem) => elem.recipient === "Для нового работника"
        );
        assistantMaterials.value = response.data.results.filter(
          (elem) => elem.recipient === "Для помощника"
        );
        bossMaterials.value = response.data.results.filter(
          (elem) => elem.recipient === "Для руководителя"
       );
      } catch (error) {
        console.error("Ошибка загрузки материалов", error);
      }
    };
 
    const resetMeetingForm = () => {
      newMeetingDateStart.value = '';
      newMeetingTimeStart.value = '';
      newMeetingDateEnd.value = '';
      newMeetingTimeEnd.value = '';
      newMeetingEmail.value = [];
      newMeetingTopic.value = 'Встреча по адаптации';
      newMeetingLocation.value = '';
      collaboratorDialogOpen.value = false;
    };
 
    const postNewMeeting = async (adaptationId) => {
      try {
        const emailsArray = newMeetingEmail.value;
 
        const requestBody = {
          action: "eval_action",
          remote_action_id: "7510212276354488535",
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
 
        const response = await axios.post(`${BACKEND_POST_URL}${queryString}`, formData, {
          headers: { "Content-Type": "multipart/form-data" },
        });
 
        showToast(response.data?.message || "Встреча успешно создана");
        // Очистка и закрытие
      newMeetingModalOpen.value = false;
      meetingModalId.value = null;
      resetMeetingForm();
      collaboratorSearch.value = "";
      fetchCollaboratorSearch;
      await fetchUsersData();
      } catch (error) {
        console.error("Ошибка при создании новой встречи", error);
        showErrorToast("Ошибка при создании новой встречи");
      }
 
      // Очистка и закрытие
      newMeetingModalOpen.value = false;
      meetingModalId.value = null;
      resetMeetingForm();
      collaboratorSearch.value = "";
      fetchCollaboratorSearch;
      await fetchUsersData();
    };
 
    const handleDialogClose = () => {
      newMeetingModalOpen.value = false;
      meetingModalId.value = null;
      resetMeetingForm();
    };
 
    const openConfirmDeleteMentor = (id) => {
      mentorToDeleteId.value = id;
      confirmDeleteMentorDialogOpen.value = true;
    };
 
    const confirmDeleteMentor = () => {
      if (mentorToDeleteId.value !== null) {
        deleteDelegationInfo(mentorToDeleteId.value);
        confirmDeleteMentorDialogOpen.value = false;
        mentorToDeleteId.value = null;
      }
    };
 
    const fetchCollaboratorList = async () => {
      try {
        const pageSize = 10;
        const page = collaboratorPage.value;
        const start = (page - 1) * pageSize;
 
        const params = {
          collection_code: "vtbl_adaptation_tutor_2025",
          secid: wtSecId,
          limit: pageSize,
          page: page,
          start: start,
          parameters: `data_mode=collaborator_list;search_str=${collaboratorSearch.value};secid=${wtSecId}`,
        };
 
        const response = await axios.post(
          BACKEND_URL,
          new URLSearchParams(params).toString()
        );
 
        collaboratorListData.value = response.data.results;
      } catch (error) {
        console.error("Ошибка при загрузке сотрудников:", error);
      }
    };
 
    const fetchCollaboratorSearch = async () => {
      try {
        const pageSize = 10;
        const page = 0;
        const start = 0;
 
        const params = {
          collection_code: "vtbl_adaptation_tutor_2025",
          secid: wtSecId,
          limit: pageSize,
          page: page,
          start: start,
          parameters: `data_mode=collaborator_list;search_str=${collaboratorSearch.value};secid=${wtSecId}`,
        };
 
        const response = await axios.post(
          BACKEND_URL,
          new URLSearchParams(params).toString()
        );
 
        collaboratorListData.value = response.data.results;
      } catch (error) {
        console.error("Ошибка при загрузке сотрудников:", error);
      }
    };
 
    const pluralizeMeetings = (count) => {
      const mod10 = count % 10;
      const mod100 = count % 100;
 
      if (mod10 === 1 && mod100 !== 11) return `${count} встреча`;
      if (mod10 >= 2 && mod10 <= 4 && !(mod100 >= 12 && mod100 <= 14)) return `${count} встречи`;
      return `${count} встреч`;
    };
 
    const changeStepperStyle = () => {
      document.querySelectorAll(".q-stepper__tab").forEach((el) => {
        el.style.paddingLeft = "5px";
        el.style.paddingRight = "5px";
      });
 
      document.querySelectorAll(".q-stepper__dot").forEach((el) => {
        el.style.background = "#00AAFF";
      });
 
      document.querySelectorAll(".q-stepper__title").forEach((el) => {
        el.style.color = "#00AAFF";
        el.style.whiteSpace = "nowrap";
      });
 
      document.querySelectorAll(".q-stepper__header").forEach((el) => {
        el.style.flexWrap = "nowrap";
      });
 
      // навешиеваем более яркий цвет на стрелки stepper
      const style = document.createElement("style");
      style.innerHTML = `
        .q-stepper--horizontal .q-stepper__line:before,
        .q-stepper--horizontal .q-stepper__line:after {
          background: rgba(0, 170, 255, 1) !important;
          height: 4px !important;
        }
      `;
      document.head.appendChild(style);
    };
 
    const changeFontSizeStyle = () => {
      document.querySelectorAll(".q-stepper__title").forEach((el) => {
        el.style.fontSize = "12px";
        el.style.display = "flex";
      });
    };
 
    const changeAnotherFontSizeStyle = () => {
      document.querySelectorAll(".q-stepper__caption").forEach((el) => {
        el.style.fontSize = "12px";
        el.style.display = "flex";
      });
    };
 
    const changeLeftLineStyle = () => {
      document.querySelectorAll(".q-item").forEach((el) => {
        el.style.paddingLeft = "0";
      });
    };
 
    const changeQItemLabelSize = () => {
      document.querySelectorAll(".q-item__label").forEach((el) => {
        el.style.fontSize = "16px";
      });
    };
 
 
 
    onMounted(async () => {
      loadExpansionState();
      fetchMaterialsData();
      fetchUsersData();
 
 
      observer = new MutationObserver(async (mutationsList) => {
        // Выполняем стилизацию при любом изменении
        mutationsList.forEach(() => {
          changeStepperStyle();
          changeFontSizeStyle();
          changeAnotherFontSizeStyle();
          changeLeftLineStyle();
          //changePrintButtonStyle();
 
          changeQItemLabelSize();
        });
      });
 
      observer.observe(document.querySelector(".content-container"), {
        childList: true,
        subtree: true,
        attributes: true, // Отслеживание изменений атрибутов
      });
    });
 
    onUnmounted(() => {
      observer.disconnect();
    });
 
    return {
      toastVisible,
      toastMessage,
      showToast,
      bossActionsData,
      usersData,
      materialsData,
      printPage,
      openMeetingModal,
      changeLeftLineStyle,
      changeQItemLabelSize,
      isMeetingModalOpen,
      personFullname,
      mentorFullname,
      userEventsLentgh,
      userSecondStage,
      userThirdStage,
      isOtherInfoOpen,
      openOtherInfo,
      appointAssistantModalId,
      isAppointAssistantOpen,
      isAppointAssistantClosing,
      openAppointAssistantModal,
      optionGroup,
      createNewMeeting,
      openDelegationError,
      closeDelegationModal,
      newMeetingModalOpen,
      newMeetingDateStart,
      newMeetingDateEnd,
      newMeetingEmail,
      newMeetingTimeStart,
      newMeetingTimeEnd,
      newMeetingTopic,
      newMeetingLocation,
      underGoingAdaptation,
      preparingForReception,
      completedAdaptation,
      wasFiredOnAdaptation,
      newEmployeeMaterials,
      assistantMaterials,
      bossMaterials,
      meetingModalId,
      currentMentor,
      currentChangeList,
      currentTaskList,
      openInfoMap,
      expansionStates,
      saveExpansionState,
      loadExpansionState,
      collaboratorListData,
      sessionID,
      collaboratorDialogOpen,
      collaboratorSearch,
      collaboratorPage,
      selectCollaborator,
      fetchCollaboratorList,
      fetchCollaboratorSearch,
      postNewMeeting,
      sendDelegationInfo,
      selectedTasks,
      selectedTaskIds,
      errorToastVisible,
      errorToastMessage,
      showErrorToast,
      deleteDelegationInfo,
      treeData,
      disabledDelegationTasks,
      pluralizeMeetings,
      handleDialogClose,
      openConfirmDeleteMentor,
      confirmDeleteMentorDialogOpen,
      confirmDeleteMentor,
      openCollaboratorDialog,
      closeCollaboratorDialog,
      newMeetingEmailInput
    };
  },
};
</script>

