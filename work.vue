<!-- Модальное окно: задачи, которые нельзя делегировать -->
<q-dialog v-model="isAppointAssistantClosing" persistent>
  <q-card class="forbidden-delegate q-pa-md" style="width: 600px; max-width: 90vw;">
    
    <!-- Верхняя панель с крестиком -->
    <div class="row justify-end">
      <q-btn
        flat
        dense
        round
        icon="close"
        color="primary"
        v-close-popup
      />
    </div>

    <!-- Заголовок -->
    <div class="text-h6 q-mb-md">
      Обратите внимание, что при делегировании остались действия,<br />
      которые необходимо выполнить непосредственному руководителю, а именно…
    </div>

    <!-- Список -->
    <q-card-section class="scroll" style="max-height: 300px;">
      <q-list bordered separator>
        <q-item
          v-for="task in disabledDelegationTasks"
          :key="task.id"
        >
          <q-item-section>{{ task.name }}</q-item-section>
        </q-item>
      </q-list>
    </q-card-section>

    <!-- Нижняя кнопка -->
    <q-card-actions align="right" class="q-pt-none">
      <q-btn
        flat
        label="Закрыть"
        color="primary"
        v-close-popup
      />
    </q-card-actions>

  </q-card>
</q-dialog>
