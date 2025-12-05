<q-btn
  class="add-task-button"
  :label="task.actualDueDate ? 'Изменить дату' : 'Добавить дату'"
  color="secondary"
  @click="openDateModal(month.title, task.id, task.actualDueDate)"
/>

<q-dialog v-model="isDateModalOpen">
  <q-card class="column" style="min-width: 500px; max-width: 600px">

    <div class="row justify-between no-wrap">
      <span style="margin-top: 20px; margin-left: 20px">
        {{ newTaskActualDate ? 'Изменить дату фактического выполнения задачи' : 'Добавить дату фактического выполнения задачи' }}
      </span>

      <q-btn icon="close" flat dense round style="margin: 10px" @click="closeDateModal" />
    </div>

    <q-card-section>
      <div class="q-gutter-md column">

        <div class="date-title row">
          <span>Текущая дата:</span>
          <span>{{ newTaskActualDate || 'Дата не выбрана' }}</span>
        </div>

        <q-input
          v-model="newTaskActualDate"
          :label="newTaskActualDate ? 'Срок выполнения (изменить)' : 'Срок выполнения (добавить)'"
          outlined
          style="max-width: 200px"
        >
          <template #append>
            <q-icon name="event" class="cursor-pointer">
              <q-popup-proxy cover transition-show="scale" transition-hide="scale">
                <q-date
                  v-model="newTaskActualDate"
                  mask="DD.MM.YYYY"
                  :locale="ruLocale"
                />
              </q-popup-proxy>
            </q-icon>
          </template>
        </q-input>

      </div>
    </q-card-section>

    <q-card-actions align="right">
      <q-btn flat label="Отмена" color="grey" @click="closeDateModal" />
      <q-btn flat label="Сохранить" color="secondary" @click="saveDateTask" />
    </q-card-actions>

  </q-card>
</q-dialog>
