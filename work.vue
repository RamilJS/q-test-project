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
