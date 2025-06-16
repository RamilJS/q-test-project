const getUserRoles = (usersString) => {
  if (!usersString) return [];
  return usersString.split(';').map((role) => {
    switch (role.trim()) {
      case 'manager':
        return 'Руководитель';
      case 'collaborator':
        return 'Сотрудник';
      case 'mentor':
        return 'Наставник';
      default:
        return role; // на случай непредвиденных ролей
    }
  });
};

const getTooltipText = (task) => {
  if (!task.users) {
    return 'Задача сохраняется автоматически';
  }

  const roles = getUserRoles(task.users);
  if (roles.length === 0) {
    return 'Задача сохраняется автоматически';
  }

  return `Данное поле заполняется ${roles.join(', ')}`;
};

const getRoleColor = (role) => {
  switch (role.trim()) {
    case 'mentor':
      return 'blue';
    case 'collaborator':
      return 'green';
    case 'manager':
      return 'red';
    default:
      return 'grey';
  }
};

<!-- Отображаем блок автора действия только если task.users не пустой -->
<div
  v-if="task.users"
  class="manager-title"
  style="
    margin-left: 10px;
    background-color: #ddd;
    border-radius: 10px;
    width: fit-content;
    padding: 0 5px;
  "
>
  <template v-for="role in task.users.split(';')" :key="role">
    <q-icon :color="getRoleColor(role)" size="8px" name="fa-solid fa-circle" />
    <span class="manager-title-item" style="margin-left: 5px; font-size: 12px;">
      {{ getUserLabel(role) }}
    </span>
  </template>
</div>

