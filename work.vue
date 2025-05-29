<!-- Удаляем старую кнопку "Отправить" в верхней части -->

<!-- Новый блок в самом низу -->
<q-card-actions align="right">
  <q-btn
    flat
    label="Закрыть"
    color="primary"
    v-close-popup
  />
  <q-btn
    label="Отправить"
    color="primary"
    @click="postNewMeeting(employee.id)"
  />
</q-card-actions>

