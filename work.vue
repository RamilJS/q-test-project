import { ref } from 'vue';
import axios from 'axios';
import { useRoute } from 'vue-router';
import { Notify } from 'quasar';

const route = useRoute();

export default {
  setup() {
    const cabinetData = ref([]);
    const newCommentText = ref('');
    const isCommentModalOpen = ref(false);
    const newCommentModalId = ref(null);
    const adaptationId = route.params.id;

    const showToast = (message, type = 'negative') => {
      Notify.create({ message, type });
    };

    // Функция для загрузки данных кабинета (остаётся с async/await)
    const fetchCabinetData = async () => {
      try {
        const params = {
          collection_code: '',
          parameters: `data_mode=collaborator_info_task_data;adaptation_id=${adaptationId}`,
        };
        const response = await axios.post(
          BACKEND_URL,
          new URLSearchParams(params).toString()
        );
        cabinetData.value = response.data.results;
        console.log('Данные кабинета загружены:', response.data); // Логируем для диагностики
      } catch (error) {
        console.error('Ошибка загрузки курса:', error.response?.data || error.message);
        showToast('Ошибка загрузки данных');
      }
    };

    // Функция для открытия модального окна добавления нового комментария
    const openCommentModal = (taskId) => {
      newCommentModalId.value = taskId;
      isCommentModalOpen.value = true;
    };

    // Функция отправки комментария (остаётся с async/await)
    const postCommentState = async (taskId) => {
      if (!newCommentText.value.trim()) {
        showToast('Введите текст комментария', 'warning');
        return;
      }

      try {
        const requestBody = {
          action: 'eval_action',
          remote_action_id: '',
          wvars: [
            { name: '_object_id', value: '' },
            { name: '_secid', value: wtSecId },
            { name: 'user_id', value: '' },
            { name: 'adaptation_id', value: String(adaptationId) },
            { name: 'role', value: 'collaborator' },
            { name: 'action_name', value: 'add_comment_task' },
            { name: 'task_id', value: taskId },
            { name: 'task_status', value: '' },
            { name: 'task_comment', value: String(newCommentText.value) },
            { name: 'comment_text', value: '' },
          ],
        };

        const formData = new FormData();
        formData.append('action', JSON.stringify(requestBody));
        const queryString = new URLSearchParams({ secid: wtSecId }).toString();

        const response = await axios.post(`${BACKEND_POST_URL}${queryString}`, formData, {
          headers: { 'Content-Type': 'multipart/form-data' },
        });

        console.log('Ответ сервера при отправке комментария:', response.data); // Логируем ответ
        newCommentText.value = '';
        isCommentModalOpen.value = false;
        newCommentModalId.value = null;
      } catch (error) {
        console.error('Ошибка при добавлении комментария:', error.response?.data || error.message);
        showToast('Ошибка при добавлении комментария');
        throw error; // Пробрасываем ошибку для обработки в submitComment
      }
    };

    // Функция отправки комментария с последующим обновлением данных (использует колбэк)
    const submitComment = (taskId, callback) => {
      postCommentState(taskId)
        .then(() => {
          fetchCabinetData()
            .then(() => {
              showToast('Комментарий успешно добавлен', 'positive');
              if (callback) callback(null); // Вызываем колбэк при успехе
            })
            .catch((error) => {
              console.error('Ошибка при обновлении данных:', error);
              showToast('Ошибка при обновлении данных');
              if (callback) callback(error); // Передаём ошибку в колбэк
            });
        })
        .catch((error) => {
          // Ошибка уже обработана в postCommentState
          if (callback) callback(error); // Передаём ошибку в колбэк
        });
    };

    return {
      cabinetData,
      newCommentText,
      isCommentModalOpen,
      newCommentModalId,
      openCommentModal,
      submitComment,
    };
  },
};
