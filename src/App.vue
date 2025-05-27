const copyToClipboard = async (taskId, path) => {
  try {
    await navigator.clipboard.writeText(path);

    // Обновляем реактивно
    copiedState.value = { ...copiedState.value, [taskId]: true };

    setTimeout(() => {
      copiedState.value = { ...copiedState.value, [taskId]: false };
    }, 3000);
  } catch (err) {
    console.error("Ошибка при копировании: ", err);
  }
};
