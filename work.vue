const resetMeetingForm = () => {
  newMeetingDateStart.value = '';
  newMeetingTimeStart.value = '';
  newMeetingDateEnd.value = '';
  newMeetingTimeEnd.value = '';
  newMeetingEmail.value = '';
  newMeetingTopic.value = 'Встреча по адаптации';
  newMeetingLocation.value = '';
  collaboratorDialogOpen.value = false;
  selectedCollaborator.value = null;
};

const postNewMeeting = async (employeeId) => {
  try {
    // отправка данных
    await axios.post('/your-endpoint', {
      // данные встречи
    });

    // закрыть диалог и сбросить форму
    newMeetingModalOpen.value = false;
    meetingModalId.value = null;
    resetMeetingForm();

  } catch (error) {
    console.error('Ошибка при отправке:', error);
  }
};

<q-btn
  flat
  label="Закрыть"
  color="primary"
  @click="handleDialogClose"
/>
const handleDialogClose = () => {
  newMeetingModalOpen.value = false;
  meetingModalId.value = null;
  resetMeetingForm();
};
