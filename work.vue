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
