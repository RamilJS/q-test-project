<li class="table-row-item actual-date-cell">
  <div class="column justify-center full-height">

    <!-- Показать дату, если она есть -->
    <span
      v-if="task.actualDueDate"
      class="actual-date-text"
    >
      {{ task.actualDueDate }}
    </span>

    <!-- Кнопка, когда даты НЕТ -->
    <q-btn
      v-else
      class="add-task-button"
      label="Добавить дату"
      color="secondary"
      size="sm"
      dense
      flat
      @click="openDateModal(month.title, task.id, null)"
    />

    <!-- Кнопка "Изменить дату", когда есть дата -->
    <q-btn
      v-if="task.actualDueDate"
      class="add-task-button edit-button"
      label="Изменить"
      color="secondary"
      size="sm"
      dense
      flat
      @click="openDateModal(month.title, task.id, task.actualDueDate)"
    />
  </div>
</li>
.table-row-item {
  height: 40px; /* высота строки таблицы */
}

.actual-date-cell {
  display: flex;
  align-items: center;
  padding: 0 8px;
}

.full-height {
  height: 100%;
}

.actual-date-text {
  font-size: 14px;
  line-height: 40px;  /* дата выравнивается по центру строки */
}

.add-task-button {
  align-self: flex-start;
  margin-top: 6px; /* чтобы кнопка стояла точно там же, где дата */
  font-size: 12px !important;
  padding: 4px 10px !important;
}

.edit-button {
  margin-top: 4px;
}
