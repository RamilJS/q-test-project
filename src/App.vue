<!--Блок-руководитель с переключателем состояния-->
<div class="main-toggle-container">
  <!--Флаг автор действия руководитель или наставник-->
  <div
    class="manager-title"
    style="
      margin-left: 10px;
      background-color: #ddd;
      border-radius: 10px;
      width: fit-content;
      padding: 0 5px;
    ">
    <q-icon
      :color="(task.description === 'Получите от руководителя задачи на испытательный срок' || process.title === 'Программа обучения') 
              ? 'green' 
              : 'red'"
      size="8px"
      name="fa-solid fa-circle"
    />
    <q-icon
      v-if="false"
      color="blue"
      size="8px"
      name="fa-solid fa-circle"
    />
    <!--Тут либо руководитель будет либо помощник по адаптации-->
    <span
      class="manager-title-item"
      style="margin-left: 5px; font-size: 12px;"
    >
      {{ (task.description === 'Получите от руководителя задачи на испытательный срок' || process.title === 'Программа обучения') 
          ? 'Сотрудник' 
          : 'Руководитель' }}
    </span>
    <span
      v-if="false"
      class="manager-title-item"
      style="margin-left: 5px; font-size: 12px;"
    >
      Помощник
    </span>
  </div>
  <!-- ... остальной код переключателя ... -->
</div>
