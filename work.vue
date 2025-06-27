<span
  class="manager-title-item"
  style="margin-left: 5px; font-size: 12px;"
>
  {{ formatUserRoles(task.users) }}
</span>

const formatUserRoles = (usersString) => {
  const roles = getUserRoles(usersString);
  return roles.length > 1 ? roles.join(' / ') : roles[0] || '';
};

const getTooltipText = (task) => {
  const roles = getUserRoles(task.users);
  if (roles.length === 0) {
    return 'Задача сохраняется автоматически';
  }

  return `Данное поле заполняет ${roles.length > 1 ? roles.join(' / ') : roles[0]}`;
};
