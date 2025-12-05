<li class="table-row-item actual-date-cell">
  <div class="row items-center justify-between no-wrap" style="width: 100%">
    <span class="actual-date-text">
      {{ task.actualDueDate || '—' }}
    </span>

    <q-btn
      class="add-task-button"
      :label="task.actualDueDate ? 'Изменить' : 'Добавить'"
      color="secondary"
      size="sm"
      dense
      flat
      @click="openDateModal(month.title, task.id, task.actualDueDate)"
    />
  </div>
</li>

.actual-date-cell {
  display: flex;
  align-items: center;
}

.actual-date-text {
  font-size: 14px;
  color: #333;
}

.add-task-button {
  font-size: 12px !important;
  padding: 4px 10px !important;
  min-width: 90px;
  text-transform: none;
}
