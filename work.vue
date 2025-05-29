import { ref, watch } from 'vue'

export default {
  setup() {
    const newMeetingDateStart = ref('');
    const newMeetingTimeStart = ref('');
    const newMeetingDateEnd = ref('');
    const newMeetingTimeEnd = ref('');

    // Автоматическая дата окончания — копируем дату начала
    watch(newMeetingDateStart, (newDate) => {
      newMeetingDateEnd.value = newDate;
    });

    // Автоматическое время окончания — +1 час к началу
    watch(newMeetingTimeStart, (newTime) => {
      if (!newTime) return;

      const [hours, minutes] = newTime.split(':').map(Number);
      let newHours = hours + 1;

      // Учитываем переход через 23:00 -> 00:00
      if (newHours >= 24) newHours -= 24;

      // Форматируем обратно в строку с ведущим нулем
      const formattedHours = String(newHours).padStart(2, '0');
      const formattedMinutes = String(minutes).padStart(2, '0');
      newMeetingTimeEnd.value = `${formattedHours}:${formattedMinutes}`;
    });

    return {
      newMeetingDateStart,
      newMeetingTimeStart,
      newMeetingDateEnd,
      newMeetingTimeEnd
    }
  }
}
