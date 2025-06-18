const isCoworkersSuccessModalOpen = ref(false);

<q-dialog v-model="isCoworkersSuccessModalOpen">
  <q-card style="min-width: 500px">
    <q-card-section class="text-h6">
      {{ coworkersMessage }}
    </q-card-section>

    <q-card-actions align="right">
      <q-btn label="Закрыть" flat color="primary" @click="isCoworkersSuccessModalOpen = false" />
      <q-btn
        label="Порекомендовать ещё коллегу"
        color="primary"
        @click="recommendAnotherCoworker"
      />
    </q-card-actions>
  </q-card>
</q-dialog>

const recommendAnotherCoworker = async () => {
  isCoworkersSuccessModalOpen.value = false;
  await nextTick(); // чтобы гарантировать закрытие предыдущего окна
  openCoworkersModal(); // снова открываем форму
};

saveCoworkersRecommendation()
newComment.value.text = "";
selectedCoworkerId.value = null;
collaboratorSearch.value = "";
isCoworkersModalOpen.value = false;
await fetchCollaboratorSearch();
isCoworkersSuccessModalOpen.value = true;
