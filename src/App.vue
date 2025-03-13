<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<q-expansion-item
  v-for="month in cabinetData"
  :key="month.id"
  :label="month.title"
  :expanded="expandedTaskProcesses[month.id]"
  @click="toggleMonthExpanded(month.id)"
>
  <!-- Контент месяца -->
</q-expansion-item>

<script>
import { ref, nextTick, onMounted, onUnmounted } from "vue";
import { useQuasar, date } from "quasar";
import axios from "axios";

const BACKEND_URL = window.location.origin + `/pp/Ext5/extjs_json_collection_data.html`;
const BACKEND_POST_URL = window.location.origin + `/lpapi.html?object_id=&doc_id=&`;

export default {
  setup() {
    const toastVisible = ref(false);
    const toastMessage = ref("");
    const newTaskText = ref("");
    const newTaskDate = ref("");
    const selectedMonth = ref(null);
    const isModalOpen = ref(false);
    const cabinetData = ref([]);
    const usersData = ref([]);
    const personFullname = ref("");
    const mentorFullname = ref("");
    const userEventsLentgh = ref(null);
    const userSecondStage = ref(null);
    const userThirdStage = ref(null);
    const allTaskExpanded = ref(false);
    const expandedTaskProcesses = ref({}); // Состояния по каждому месяцу
    let observer;

    // Показ уведомлений
    const showToast = (message = "Задача сохранена") => {
      toastMessage.value = message;
      toastVisible.value = true;
      setTimeout(() => {
        toastVisible.value = false;
      }, 2000);
    };

    // Модалка
    const openModal = (monthTitle) => {
      selectedMonth.value = monthTitle;
      isModalOpen.value = true;
    };
    const closeModal = () => {
      isModalOpen.value = false;
    };
    const clearModal = () => {
      newTaskText.value = "";
      newTaskDate.value = "";
      isModalOpen.value = false;
    };

    const fetchCabinetData = async () => {
      try {
        const params = {
          collection_code: "vtbl_adaptation_2025",
          parameters: "data_mode=collaborator_result_task_data",
        };

        const response = await axios.post(
          BACKEND_URL,
          new URLSearchParams(params).toString()
        );

        cabinetData.value = response.data.results.map((month) => ({
          ...month,
          tasks: month.tasks.filter((task) => !task.isTemporary),
        }));
      } catch (error) {
        console.error("Ошибка загрузки данных", error);
      }
    };

    const fetchUsersData = async () => {
      try {
        const params = {
          collection_code: "vtbl_adaptation_2025",
          parameters: "data_mode=get_adaptation_info",
        };
        const response = await axios.post(
          BACKEND_URL,
          new URLSearchParams(params).toString()
        );

        usersData.value = response.data.results[0];
        personFullname.value = response.data.results[0].person.person_fullname;
        mentorFullname.value = response.data.results[0].mentor.person_fullname;
        userEventsLentgh.value = response.data.results[0].events.length;
        userSecondStage.value = response.data.results[0].stages[1].percent;
        userThirdStage.value = response.data.results[0].stages[2].percent;
      } catch (error) {
        console.error("Ошибка загрузки кабинета", error);
      }
    };

    // NEW: Сохранение состояния всех месяцев
    const saveExpandedStates = () => {
      sessionStorage.setItem('expandedTaskProcesses', JSON.stringify(expandedTaskProcesses.value));
    };

    // При инициализации проверяем сохранённое значение
    const savedAllTaskExpanded = sessionStorage.getItem('allTaskExpanded');
    if (savedAllTaskExpanded !== null) {
      allTaskExpanded.value = JSON.parse(savedAllTaskExpanded);
    }

    // NEW: Тогглим конкретный месяц и сохраняем
    const toggleMonthExpanded = (monthId) => {
      expandedTaskProcesses.value[monthId] = !expandedTaskProcesses.value[monthId];
      saveExpandedStates(); // сохраняем состояние после изменения
    };

    // Раскрыть / свернуть все месяцы
    const toggleProcessList = () => {
      if (allTaskExpanded.value) {
        cabinetData.value.forEach((month) => {
          expandedTaskProcesses.value[month.id] = false;
        });
      } else {
        cabinetData.value.forEach((month) => {
          expandedTaskProcesses.value[month.id] = true;
        });
      }

      allTaskExpanded.value = !allTaskExpanded.value;

      sessionStorage.setItem('allTaskExpanded', JSON.stringify(allTaskExpanded.value));
      saveExpandedStates(); // NEW: сохраняем состояния по каждому месяцу
    };

    const updateMonthState = (month) => {
      month.state = month.tasks.every(task => task.completed) ? 'Выполнено' : 'Не выполнено';
    };

    const updateLocalTask = (taskId, newValue) => {
      cabinetData.value.forEach((process) => {
        process.tasks.forEach((task) => {
          if (task.id === taskId) {
            task.completed = newValue;
          }
        });
      });
    };

    const postTaskState = async (newValue, taskId) => {
      try {
        const requestBody = {
          action: "eval_action",
          remote_action_id: "7472730402329554549",
          wvars: [
            { name: "_object_id", value: "" },
            { name: "_secid", value: wtSecId },
            { name: "user_id", value: "" },
            { name: "adaptation_id", value: "" },
            { name: "role", value: "collaborator" },
            { name: "action_name", value: "save_task" },
            { name: "task_id", value: taskId },
            { name: "task_status", value: Number(newValue) },
            { name: "task_comment", value: "" },
          ],
        };

        const formData = new FormData();
        formData.append("action", JSON.stringify(requestBody));

        const queryString = new URLSearchParams({ secid: wtSecId }).toString();

        await axios.post(`${BACKEND_POST_URL}${queryString}`, formData, {
          headers: { "Content-Type": "multipart/form-data" },
        });

      } catch (error) {
        console.error("Ошибка при обновлении состояния задачи", error);
        showToast("Ошибка при обновлении задачи");
      }
    };

    const handleToggle = async (newValue, id, month) => {
      await postTaskState(newValue, id);
      updateLocalTask(id, newValue);
      updateMonthState(month);
      showToast("Задача сохранена");
    };

    const updateLocalNewTaskList = (taskMonth, newTask) => {
      const month = cabinetData.value.find((m) => m.title === taskMonth);
      if (month) {
        month.tasks.push(newTask);
      } else {
        console.warn(`Не найден месяц: ${taskMonth}`);
      }
    };

    const postNewTask = async (taskMonth, taskText, taskDate) => {
      try {
        const requestBody = {
          action: "eval_action",
          remote_action_id: "7472730402329554549",
          wvars: [
            { name: "_object_id", value: "" },
            { name: "_secid", value: wtSecId },
            { name: "user_id", value: "" },
            { name: "adaptation_id", value: "" },
            { name: "role", value: "collaborator" },
            { name: "action_name", value: `add_${taskMonth}_mounth_task` },
            { name: "task_name", value: taskText },
            { name: "task_date", value: taskDate },
            { name: "task_comment", value: "" },
          ],
        };

        const formData = new FormData();
        formData.append("action", JSON.stringify(requestBody));

        const queryString = new URLSearchParams({ secid: wtSecId }).toString();

        const tempId = `temp-${Date.now()}`;

        updateLocalNewTaskList(taskMonth, {
          id: tempId,
          taskName: taskText,
          dueDate: taskDate,
          completed: false,
          canChange: true,
          completedMentor: false,
          isTemporary: true,
        });

        await axios.post(`${BACKEND_POST_URL}${queryString}`, formData, {
          headers: { "Content-Type": "multipart/form-data" },
        });

        showToast("Задача добавлена");

        fetchCabinetData();
      } catch (error) {
        console.error("Ошибка при добавлении задачи", error);
        showToast("Ошибка при добавлении задачи");
      }
    };

    const addNewTask = async (taskMonth, taskText, taskDate) => {
      await postNewTask(taskMonth, taskText, taskDate);
    };

    const saveTask = () => {
      const monthMap = {
        "Первый месяц работы": "1",
        "Второй месяц работы": "2",
        "Третий месяц работы": "3",
      };

      if (monthMap[selectedMonth.value]) {
        selectedMonth.value = monthMap[selectedMonth.value];
      } else {
        showToast("Некорректное название месяца");
        return;
      }

      if (!newTaskText.value.trim() || !newTaskDate.value.trim()) {
        showToast("Заполните все поля");
        return;
      }

      addNewTask(selectedMonth.value, newTaskText.value, newTaskDate.value);
      closeModal();
      clearModal();
    };

    const changeStyles = () => {
      document.querySelectorAll(".q-expansion-item__container").forEach((el) => {
        el.style.width = "100%";
      });
    };

    const changeStepperStyle = () => {
      document.querySelectorAll(".q-stepper__tab").forEach((el) => {
        el.style.paddingLeft = "0";
        el.style.paddingRight = "0";
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
        el.style.display = "flex";
      });
    };

    const changeLeftLineStyle = () => {
      document.querySelectorAll(".q-item").forEach((el) => {
        el.style.paddingLeft = "0";
      });
    };

    const changeQItemLabelStyle = () => {
      document.querySelectorAll(".q-item__label").forEach((el) => {
        el.style.fontWeight = "700";
      });
    };

    const changeQItemLabelSize = () => {
      document.querySelectorAll(".q-item__label").forEach((el) => {
        el.style.fontSize = "16px";
      });
    };

    const stepperTab = () => {
      document.querySelector(".q-stepper__tab").style.width = "min-content";
    };

    onMounted(async () => {
      await fetchCabinetData();
      fetchUsersData();

      // NEW: загрузка состояния каждого месяца
      const savedExpandedStates = sessionStorage.getItem('expandedTaskProcesses');
      if (savedExpandedStates) {
        expandedTaskProcesses.value = JSON.parse(savedExpandedStates);
      } else {
        cabinetData.value.forEach((month) => {
          expandedTaskProcesses.value[month.id] = allTaskExpanded.value;
        });
      }

      nextTick(() => {
        changeStyles();
        stepperTab();
      });

      observer = new MutationObserver(async (mutationsList) => {
        await nextTick();
        mutationsList.forEach(() => {
          changeStyles();
          changeStepperStyle();
          changeFontSizeStyle();
          changeAnotherFontSizeStyle();
          changeLeftLineStyle();
          changeQItemLabelStyle();
          changeQItemLabelSize();
          stepperTab();
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

    return {
      toastVisible,
      toastMessage,
      showToast,
      openModal,
      closeModal,
      clearModal,
      saveTask,
      newTaskText,
      newTaskDate,
      selectedMonth,
      isModalOpen,
      cabinetData,
      usersData,
      handleToggle,
      changeStyles,
      stepperTab,
      updateMonthState,
      personFullname,
      mentorFullname,
      userEventsLentgh,
      userSecondStage,
      userThirdStage,
      allTaskExpanded,
      expandedTaskProcesses,
      toggleProcessList,
      toggleMonthExpanded, // NEW: метод для ручного раскрытия/закрытия месяца
    };
  },
};
</script>
