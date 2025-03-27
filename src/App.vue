<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

export default {
  methods: {
    openOutlook() {
      // Дата и время в формате YYYYMMDDTHHMMSSZ (UTC)
      const start = "20240410T100000Z";
      const end = "20240410T110000Z";
      const summary = "Встреча с командой";
      const description = "Обсудим проект и планы";
      const location = "Офис, ул. Примерная, 12";

      // Формируем ICS-событие
      const icsContent = `BEGIN:VCALENDAR
VERSION:2.0
BEGIN:VEVENT
SUMMARY:${summary}
DESCRIPTION:${description}
LOCATION:${location}
DTSTART:${start}
DTEND:${end}
END:VEVENT
END:VCALENDAR`;

      // Создаём Blob (временный файл)
      const blob = new Blob([icsContent], { type: "text/calendar" });
      const url = URL.createObjectURL(blob);

      // Открываем Outlook (или другое приложение календаря)
      window.location.href = url;

      // Очищаем URL после использования
      setTimeout(() => URL.revokeObjectURL(url), 10000);
    }
  }
};
