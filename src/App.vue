<template>
  <div id="q-app">
    <router-view />
  </div>
</template>


<script>
import { ref, onMounted, onUnmounted, computed, watch } from "vue";
import axios from "axios";

const BACKEND_URL = wil`;
const BACKEND_POST_URL = window;

export default {
  setup() {
    const optionGroup = ref(null);
    const toastVisible = ref(false);
    const toastMessage = ref("");
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
    const openInfoMap = ref({}); // состояние раскрытия q-expansion-item

    const isMeetingModalOpen = ref(false);
    const newMeetingModalOpen = ref(false);
    const newMeetingEmail = ref("");
    const newMeetingDateStart = ref("");
    const newMeetingDateEnd = ref("");
    const newMeetingTimeStart = ref("");
    const newMeetingTimeEnd = ref("");
    const newMeetingTopic = ref("Встреча по адаптации");
    const newMeetingLocation = ref("");

    const underGoingAdaptation = ref([]);
    const preparingForReception = ref([]);
    const completedAdaptation = ref([]);
    const wasFiredOnAdaptation = ref([]);

    const newEmployeeMaterials = ref([]);
    const assistantMaterials = ref([]);
    const bossMaterials = ref([]);
    const bossActionsData = ref([]);

    const meetingModalId = ref(null);
    const appointAssistantModalId = ref(null);
    const currentMentor = ref(null);
    const currentChangeList = ref([]);
    const currentTaskList = ref([]);

    const printPage = async (link) => {
      console.log(link);
    };

    const openDelegationError = () => {
      isAppointAssistantClosing.value = true;
    };

    const openOtherInfo = (id) => {
      openInfoMap.value[id] = !openInfoMap.value[id];
    };

    const createNewMeeting = (id) => {
      meetingModalId.value = id;
      newMeetingModalOpen.value = true;
    };

    const openMeetingModal = () => {
      isMeetingModalOpen.value = true;
    };

    const showToast = (message = "Задача сохранена") => {
      toastMessage.value = message;
      toastVisible.value = true;
      setTimeout(() => (toastVisible.value = false), 2000);
    };

    const openAppointAssistantModal = (id) => {
      appointAssistantModalId.value = id;
      isAppointAssistantOpen.value = true;

      const employee = usersData.value.find(
        (em) => em.person.person_id === id
      );

      if (employee && employee.mentor) {
        currentMentor.value = null;
        currentChangeList.value = employee.mentor.change_list || [];
        currentTaskList.value = employee.mentor.task_list || [];

        console.log(currentTaskList.value);
      }
    };

    const fetchUsersData = async () => {
      try {
        const params = {
          collection_code: "_adaptation_manager_2025",
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
      } catch (error) {
        console.error("Ошибка загрузки кабинета руководителя", error);
      }
    };

    const fetchMaterialsData = async () => {
      try {
        const params = {
          collection_code: "_adaptation_manager_2025",
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

    const postBossActions = async (newValue, taskId) => {
      try {
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
            { name: "task_id", value: "" },
            { name: "task_status", value: "" },
            { name: "task_comment", value: "" },
          ],
        };

        const formData = new FormData();
        formData.append("action", JSON.stringify(requestBody));
        const queryString = new URLSearchParams({ secid: wtSecId }).toString();

        const response = await axios.post(
          `${BACKEND_POST_URL}${queryString}`,
          formData,
          { headers: { "Content-Type": "multipart/form-data" } }
        );

        bossActionsData.value = response.data.results;
      } catch (error) {
        console.error("Ошибка отправки информации", error);
      }
    };

    const changeStepperStyle = () => {
      document.querySelectorAll(".q-stepper__tab").forEach((el) => {
        el.style.paddingLeft = "5px";
        el.style.paddingRight = "5px";
      });
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
      const savedMap = sessionStorage.getItem("openInfoMap");
      if (savedMap) {
        openInfoMap.value = JSON.parse(savedMap);
      }

      await fetchMaterialsData();
      await fetchUsersData();

      observer = new MutationObserver(async (mutationsList) => {
        mutationsList.forEach(() => {
          changeStepperStyle();
          changeFontSizeStyle();
          changeAnotherFontSizeStyle();
          changeLeftLineStyle();
          changeQItemLabelSize();
        });
      });

      observer.observe(document.querySelector(".content-container"), {
        childList: true,
        subtree: true,
        attributes: true,
      });
    });

    onUnmounted(() => {
      observer.disconnect();
    });

    watch(openInfoMap, (newVal) => {
      sessionStorage.setItem("openInfoMap", JSON.stringify(newVal));
    }, { deep: true });

    return {
      toastVisible,
      toastMessage,
      showToast,
      bossActionsData,
      postBossActions,
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
      openInfoMap
    };
  },
};
</script>
