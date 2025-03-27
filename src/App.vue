<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

export default {
  methods: {
    openOutlook() {
      const url = "https://outlook.office.com/calendar/action/compose?subject=Встреча%20с%20командой&startdt=2024-04-10T10:00:00Z&enddt=2024-04-10T11:00:00Z&location=Офис&body=Обсудим%20проект";
      window.open(url, "_blank");
    }
  }
};

<a href="ms-outlook://calendar/action/compose?subject=Встреча%20с%20командой&startdt=2024-04-10T10:00:00Z&enddt=2024-04-10T11:00:00Z&location=Офис&body=Обсудим%20проект">
  Открыть в Outlook (десктоп)
</a>
