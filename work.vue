<!--Модальное окно Предупреждение о задачах, которые нельзя делегировать-->
<q-dialog v-model="isAppointAssistantClosing">
  <q-card class="forbidden-delegate">
    <!-- Кнопка закрытия в правом верхнем углу -->
    <q-card-section class="q-pa-none">
      <q-btn
        flat
        icon="close"
        color="primary"
        v-close-popup
        class="absolute-top-right q-ma-sm"
        style="z-index: 1;"
      />
    </q-card-section>

    <q-card-section class="q-pt-xl">
      <div class="text-h6">
        Обратите внимание, что при делегировании остались действия,
        <br/>
        которые необходимо выполнить непосредственному руководителю, а именно…
      </div>
    </q-card-section>

    <q-card-section
      class="column"
      style="
        justify-content: space-between;
        padding-bottom: 0;
        flex-wrap: nowrap;
      "
    >
      <q-card-section class="scroll q-py-sm">
        <q-list bordered separator>
          <q-item v-for="task in disabledDelegationTasks" :key="task.id" style="margin-left: 10px;">
            <q-item-section>{{ task.name }}</q-item-section>
          </q-item>
        </q-list>
      </q-card-section>

      <q-btn
        flat
        label="Закрыть"
        color="primary"
        v-close-popup
        style="margin-left: auto;"
      />
    </q-card-section>
  </q-card>
</q-dialog>
