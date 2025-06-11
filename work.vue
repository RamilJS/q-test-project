<!-- Отображение роли по task.users -->
<div
  v-if="task.users && task.users.length"
  class="manager-title"
  style="
    margin-left: 10px;
    background-color: #ddd;
    border-radius: 10px;
    width: fit-content;
    padding: 0 5px;
  "
>
  <q-icon
    :color="getUserRoleColor(task.users)"
    size="8px"
    name="fa-solid fa-circle"
  />
  <span
    class="manager-title-item"
    style="margin-left: 5px; font-size: 12px;"
  >
    {{ getUserRoleName(task.users) }}
  </span>
</div>

const getUserRoleColor = (users) => {
  if (!users || !users.length) return null;

  const role = users[0]; // если всегда один, иначе добавить проверку по типу
  switch (role) {
    case 'collaborator':
      return 'green';
    case 'manager':
      return 'red';
    case 'mentor':
      return 'blue';
    default:
      return null;
  }
};

const getUserRoleName = (users) => {
  if (!users || !users.length) return '';

  const role = users[0];
  switch (role) {
    case 'collaborator':
      return 'Сотрудник';
    case 'manager':
      return 'Руководитель';
    case 'mentor':
      return 'Наставник';
    default:
      return '';
  }
};
