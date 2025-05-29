const confirmDeleteMentorDialogOpen = ref(false);
const mentorToDeleteId = ref(null);

<q-btn
  v-if="employee.mentor.person_fullname"
  label="Удалить помощника"
  flat
  color="primary"
  class="delete-mentor-button"
  size="sm"
  padding="sm"
  @click="openConfirmDeleteMentor(employee.id)"
  style="min-width: 145px;"
/>

const openConfirmDeleteMentor = (id) => {
  mentorToDeleteId.value = id;
  confirmDeleteMentorDialogOpen.value = true;
};

<q-dialog v-model="confirmDeleteMentorDialogOpen">
  <q-card style="min-width: 400px;">
    <q-card-section class="text-h6">
      Удалить помощника?
    </q-card-section>

    <q-card-actions align="right">
      <q-btn
        flat
        label="Нет"
        color="primary"
        @click="confirmDeleteMentorDialogOpen = false"
      />
      <q-btn
        label="Да"
        color="primary"
        @click="confirmDeleteMentor"
      />
    </q-card-actions>
  </q-card>
</q-dialog>

const confirmDeleteMentor = () => {
  if (mentorToDeleteId.value !== null) {
    deleteDelegationInfo(mentorToDeleteId.value);
    confirmDeleteMentorDialogOpen.value = false;
    mentorToDeleteId.value = null;
  }
};

return {
  openConfirmDeleteMentor,
  confirmDeleteMentorDialogOpen,
  confirmDeleteMentor,
};
