<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<q-dialog
  v-if="meetingModalId === employee.id"
  v-model="newMeetingModalOpen"
>
  <q-card
    class="q-pa-md"
    style="
      min-width: 650px;
      max-width: 800px;
      box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
      border-radius: 12px;
    "
  >
    <q-card-section class="q-pb-none">
      <div class="text-h6 q-mb-md text-primary">Назначение встречи</div>
    </q-card-section>

    <q-separator spaced />

    <q-card-section>
      <div class="q-gutter-md column">

        <div class="row items-center justify-between">
          <q-btn
            icon="send"
            label="Отправить"
            color="primary"
            outline
            padding="sm md"
            size="sm"
          />
        </div>

        <div class="row items-center q-gutter-sm">
          <div class="text-subtitle2" style="width: 80px;">Кому:</div>
          <q-input
            dense
            filled
            v-model="newMeetingEmail"
            type="email"
            class="col"
          />
        </div>

        <div class="row items-center q-gutter-sm">
          <div class="text-subtitle2" style="width: 80px;">Тема:</div>
          <q-input
            dense
            filled
            v-model="newMeetingTopic"
            placeholder="Встреча по адаптации"
            class="col"
          />
        </div>

        <div class="row items-center q-gutter-sm">
          <div class="text-subtitle2" style="width: 80px;">Место:</div>
          <q-input
            dense
            filled
            v-model="newMeetingLocation"
            placeholder="Укажите место"
            class="col"
          />
        </div>

        <div class="row items-center q-gutter-md">
          <div class="text-subtitle2" style="width: 130px;">Время начала:</div>
          <q-input
            dense
            filled
            v-model="newMeetingDate"
            type="date"
            style="max-width: 160px;"
          />
          <q-input
            dense
            filled
            v-model="newMeetingTimeStart"
            type="time"
            style="max-width: 100px;"
          />
        </div>

        <div class="row items-center q-gutter-md">
          <div class="text-subtitle2" style="width: 130px;">Окончание:</div>
          <q-input
            dense
            filled
            v-model="newMeetingDate"
            type="date"
            style="max-width: 160px;"
          />
          <q-input
            dense
            filled
            v-model="newMeetingTimeEnd"
            type="time"
            style="max-width: 100px;"
          />
        </div>
      </div>
    </q-card-section>

    <q-card-actions align="right">
      <q-btn
        flat
        label="Закрыть"
        color="primary"
        v-close-popup
      />
    </q-card-actions>
  </q-card>
</q-dialog>
