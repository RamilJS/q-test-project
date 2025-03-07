<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

import { ref, computed, onMounted, watchEffect, nextTick } from "vue";
import axios from "axios";

const BACKEND_URL = window.location.origin + `/pp/Ext5/extjs_json_collection_data.html`;
const BACKEND_POST_URL = window.location.origin + `/lpapi.html?object_id=&doc_id=&`;

export default {
  setup() {
    const toastVisible = ref(false);
    const toastMessage = ref("");
    const copied = ref(false);
    const cabinetData = ref([]);
    const usersData = ref([]);
    const currentCategoryTitle = ref("");
    const toggleValue = ref("Выполнено");
    const step = ref(1);

    const originURL = computed(() => `${window.location.origin}`);

    const copyToClipboard = async (link) => {
      try {
        await navigator.clipboard.writeText(link);
        copied.value = true;
        setTimeout(() => (copied.value = false), 3000);
      } catch (err) {
        console.error("Ошибка при копировании: ", err);
      }
    };

    const showToast = (message = "Задача сохранена") => {
      toastMessage.value = message;
      toastVisible.value = true;
      setTimeout(() => (toastVisible.value = false), 2000);
    };

    const fetchCabinetData = async () => {
      try {
        const params = {
          collection_code: "vtbl_adaptation_2025",
          parameters: "data_mode=collaborator_info_task_data",
        };
        const response = await axios.post(BACKEND_URL, new URLSearchParams(params).toString());
        cabinetData.value = response.data.results;
      } catch (error) {
        console.error("Ошибка загрузки курса", error);
      }
    };

    const fetchUsersData = async () => {
      try {
        const params = {
          collection_code: "vtbl_adaptation_2025",
          parameters: "data_mode=get_adaptation_info",
        };
        const response = await axios.post(BACKEND_URL, new URLSearchParams(params).toString());
        usersData.value = response.data.results[0];
      } catch (error) {
        console.error("Ошибка загрузки кабинета", error);
      }
    };

    const handleToggle = async (newValue, id) => {
      await postTaskState(newValue, id);
      updateLocalTask(id, newValue);
      showToast("Задача сохранена");
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

    onMounted(() => {
      fetchCabinetData();
      fetchUsersData();

      const observer = new MutationObserver(async () => {
        await nextTick();
        applyStyles();
      });

      observer.observe(document.body, { childList: true, subtree: true, attributes: true });
    });

    const applyStyles = () => {
      document.querySelectorAll(".q-stepper__tab").forEach((el) => {
        el.style.paddingLeft = "10px";
        el.style.paddingRight = "10px";
      });
      document.querySelectorAll(".q-expansion-item__container").forEach((el) => {
        el.style.width = "100%";
      });
      document.querySelectorAll(".q-stepper__title, .q-stepper__caption").forEach((el) => {
        el.style.display = "flex";
      });
      document.querySelectorAll(".q-item").forEach((el) => {
        el.style.paddingLeft = "0";
      });
      document.querySelectorAll(".q-item__label").forEach((el) => {
        el.style.fontWeight = "700";
        el.style.fontSize = "16px";
      });
    };

    return {
      currentCategoryTitle,
      originURL,
      toggleValue,
      step,
      showToast,
      toastVisible,
      toastMessage,
      copied,
      copyToClipboard,
      cabinetData,
      usersData,
      fetchCabinetData,
      fetchUsersData,
      handleToggle,
    };
  },
};
