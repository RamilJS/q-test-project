// ref для массива email
const newMeetingEmail = ref([]);

// ref для выбранных сотрудников (если нужно)
const selectedCollaborators = ref([]);

// выбор сотрудника
const selectCollaborator = (item) => {
  // проверяем, что email есть и его еще нет в массиве
  if (item.email && !newMeetingEmail.value.includes(item.email)) {
    newMeetingEmail.value.push(item.email);
    selectedCollaborators.value.push(item); // если нужен объект сотрудника
  }

  collaboratorDialogOpen.value = false;
};
