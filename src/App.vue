<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<div class="metting-link-wrapper column">
  <q-btn
    class="new-metting-link"
    color="primary"
    text-color="white"
    label="НАЗНАЧИТЬ ВСТРЕЧУ"
    size="sm"
    padding="sm"
    @click="createNewMeeting"
  />

  <q-dialog v-model="newMeetingModalOpen">
    <q-card class="q-pa-md" style="min-width: 450px; border-radius: 16px; box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);">
      <q-card-section class="q-pb-none">
        <div class="text-h6" style="padding-bottom: 10px;">
          Назначение встречи
        </div>
      </q-card-section>

      <q-card-section>
        <div class="column q-gutter-lg" style="max-width: 400px;">
          <div class="row items-center justify-between q-gutter-sm">
            <span>Кому</span>
            <q-input dense v-model="newMeetingEmail" filled type="email" hint="Email" />
          </div>

          <div class="row items-center justify-between q-gutter-sm">
            <span>Тема встречи</span>
            <q-input dense outlined v-model="newMeetingTopic" placeholder="Встреча по адаптации" hint="Скорректируйте тему при необходимости" />
          </div>

          <div class="row items-center justify-between q-gutter-sm">
            <span>Дата встречи</span>
            <q-input dense v-model="newMeetingDate" filled type="date" hint="Выберите дату" />
          </div>

          <div class="row items-center justify-between q-gutter-sm">
            <span>Место встречи</span>
            <q-input dense v-model="newMeetingLocation" filled hint="Укажите место встречи" />
          </div>

          <div class="row items-center justify-between q-gutter-sm">
            <span>Начало встречи</span>
            <q-input dense v-model="newMeetingTimeStart" filled type="time" hint="Выберите время" />
          </div>

          <div class="row items-center justify-between q-gutter-sm">
            <span>Конец встречи</span>
            <q-input dense v-model="newMeetingTimeEnd" filled type="time" hint="Выберите время" />
          </div>
        </div>
      </q-card-section>

      <q-card-actions align="right">
        <q-btn flat label="Закрыть" color="primary" v-close-popup />
      </q-card-actions>
    </q-card>
  </q-dialog>
</div>
