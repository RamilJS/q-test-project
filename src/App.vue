<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

export default {
  methods: {
    downloadICS() {
      // Дата и время события в формате YYYYMMDDTHHMMSSZ (UTC)
      const start = "20240410T100000Z"; // 10 апреля 2024, 10:00 UTC
      const end = "20240410T110000Z";   // 10 апреля 2024, 11:00 UTC
      const summary = "Встреча с командой";
      const description = "Обсудим проект и планы";
      const location = "Офис, ул. Примерная, 12";

      // Формируем содержимое .ics файла
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

      // Создаем Blob и скачиваем файл
      const blob = new Blob([icsContent], { type: "text/calendar" });
      const url = URL.createObjectURL(blob);

      const a = document.createElement("a");
      a.href = url;
      a.download = "meeting.ics";
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      URL.revokeObjectURL(url);
    }
  }
};
