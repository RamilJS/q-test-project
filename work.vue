
const submitComment = async (taskId) => {
  try {
    await postCommentState(taskId);
    await fetchCabinetData();
    showToast('Комментарий успешно добавлен', 'positive');
  } catch (error) {
    showToast('Ошибка при отправке комментария или обновлении данных', 'negative');
    console.error('Ошибка в submitComment:', error);
  }
};
