<div class="row justify-between items-center q-mt-sm">
  <q-btn
    flat
    icon="fa-solid fa-arrow-left"
    :disable="collaboratorPage <= 1"
    @click="collaboratorPage-- && fetchCollaboratorList()"
  />

  <q-btn
    flat
    icon="fa-solid fa-arrow-right"
    @click="collaboratorPage++ && fetchCollaboratorList()"
  />
</div>
