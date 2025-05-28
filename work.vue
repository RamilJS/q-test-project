<q-tooltip class="bg-white text-black">
  <ul class="q-mb-none q-pa-none">
    <li
      v-for="event in employee.events"
      :key="event.date_start"
    >
      {{ event.date_start }} — {{ event.subject_line || 'Без темы' }}
    </li>
  </ul>
</q-tooltip>


<q-btn
  v-if="employee.events.length > 0"
  color="grey"
  icon="fa-solid fa-info"
  size="6px"
  push
  round
  class="metting-modal-button"
  style="margin-bottom: auto; margin-left: 10px;"
>
  <q-tooltip class="bg-white text-black" style="white-space: pre-line;">
    {{ formattedEvents }}
  </q-tooltip>
</q-btn>
const formattedEvents = computed(() =>
  employee.events
    .map(event => `• ${event.date_start} — ${event.subject_line || 'Без темы'}`)
    .join('\n')
);
