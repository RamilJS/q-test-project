const saveTask = () => {
  const monthMap = {
    "Первый месяц работы": "1",
    "Второй месяц работы": "2",
    "Третий месяц работы": "3",
  };

  const rawMonth = selectedMonth.value;
  const mappedMonth = monthMap[rawMonth];

  if (!mappedMonth) {
    showToast("Некорректное название месяца");
    return;
  }

  if (!newTaskText.value.trim() || !newTaskDate.value) {
    showToast("Заполните все поля");
    return;
  }

  // Дополнительная валидация даты
  const parsedDate = new Date(newTaskDate.value);
  if (isNaN(parsedDate.getTime())) {
    showToast("Некорректная дата");
    return;
  }

  addNewTask(mappedMonth, newTaskText.value, newTaskDate.value);
  closeModal();
  clearModal();
};

const clearModal = () => {
  newTaskText.value = "";
  newTaskDate.value = "";
  selectedMonth.value = "";
  isModalOpen.value = false;
};
