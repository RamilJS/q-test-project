<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

const submitComment = async () => {
  if (!newComment.value.text.trim() || newComment.value.text.length > 1000)
    return;

  try {
    await postCommentState(newComment.value.text);
    await fetchCommentListData(); // <-- вместо локального добавления
    newComment.value.text = "";
  } catch (error) {
    console.error("Ошибка при отправке комментария", error);
    showToast("Не удалось отправить комментарий");
  }
};
