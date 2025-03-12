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

<script setup>
import { ref, computed } from 'vue'

// Пример списка комментариев
const commentListData = ref([
  {
    id: 1,
    date: '19.02.2025 00:00:00',
    person_fullname: 'Иванов Иван',
    person_position: 'Менеджер',
    person_subdivision: 'Отдел продаж',
    person_icon_url: 'https://example.com/avatar1.jpg',
    text: 'Комментарий 1'
  },
  {
    id: 2,
    date: '10.01.2024 15:20:30',
    person_fullname: 'Петров Петр',
    person_position: 'Разработчик',
    person_subdivision: 'IT отдел',
    person_icon_url: 'https://example.com/avatar2.jpg',
    text: 'Комментарий 2'
  }
  // другие комментарии
])

// Функция для парсинга даты
const parseDate = (dateStr) => {
  const [datePart, timePart] = dateStr.split(' ')
  const [day, month, year] = datePart.split('.').map(Number)
  const [hours, minutes, seconds] = timePart.split(':').map(Number)
  return new Date(year, month - 1, day, hours, minutes, seconds)
}

// Вычисляемое свойство для сортировки комментариев
const sortedCommentList = computed(() => {
  return [...commentListData.value].sort((a, b) => {
    const dateA = parseDate(a.date)
    const dateB = parseDate(b.date)
    return dateB - dateA // Сортировка по убыванию (новые выше)
  })
})
</script>
<ul class="comment-list">
  <li
    v-for="comment in sortedCommentList"
    :key="comment.id"
    class="comment-item"
  >
    <div class="comment-user-info">
      <q-avatar size="40px" class="comment-avatar">
        <img
          :src="comment.person_icon_url"
          alt="Фото сотрудника"
          style="object-fit: cover"
        />
      </q-avatar>
      <div class="user-meta">
        <div class="user-name">
          {{ comment.person_fullname }}
        </div>
        <div class="user-role">
          {{ comment.person_position }}
        </div>
        <div class="user-department">
          {{ comment.person_subdivision }}
        </div>
      </div>
      <div class="comment-date">
        <div class="comment-phone">
          {{ comment.date }}
        </div>
      </div>
    </div>
    <div class="comment-value">
      {{ comment.text }}
    </div>
  </li>
</ul>

gghhh
const allExpanded = ref(false);

// При инициализации проверяем сохранённое значение
const savedAllExpanded = sessionStorage.getItem('allExpanded');
if (savedAllExpanded !== null) {
  allExpanded.value = JSON.parse(savedAllExpanded);
}

const toggleProcessList = () => {
  if (allExpanded.value) {
    cabinetData.value.forEach((process) => {
      expandedProcesses.value[process.id] = false;
    });
  } else {
    cabinetData.value.forEach((process) => {
      expandedProcesses.value[process.id] = true;
    });
  }

  allExpanded.value = !allExpanded.value;

  sessionStorage.setItem('allExpanded', JSON.stringify(allExpanded.value));
};

onMounted(async () => {
  await fetchCabinetData();
  fetchMaterialsData();
  fetchCommentListData();
  fetchUsersData();
  fetchCoworkers();

  nextTick(() => {
    cabinetData.value.forEach((process) => {
      expandedProcesses.value[process.id] = allExpanded.value;
    });
  });
});

