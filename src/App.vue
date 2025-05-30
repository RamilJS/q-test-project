
<q-btn
  v-if="employee.mentor.person_fullname && (employee.status == 'active' || employee.status == 'planned')"
  label="Удалить помощника"
  flat
  color="primary"
  class="delete-mentor-button"
  size="sm"
  padding="sm"
  @click="openConfirmDeleteMentor(employee.id)"
  style="min-width: 145px;"
/>

<q-btn
  v-if="employee.status == 'active' || employee.status == 'planned'"
  color="primary"
  text-color="white"
  label="Назначить помощника"
  size="sm"
  padding="sm"
  @click="openAppointAssistantModal(employee.person.person_id)"
/>
