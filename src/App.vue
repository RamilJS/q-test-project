<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<!-- Поле поиска коллег -->
<div class="employee-search-container">
  <span class="text-bold">Рекомендации коллег</span>

  <q-input
    v-model="collaboratorSearch"
    @update:model-value="fetchCollaboratorSearch"
    dense
    outlined
    placeholder="Поиск"
    class="bg-white col"
    style="margin-top: 8px"
  />
</div>

<!-- Список рекомендованных коллег -->
<div class="q-pa-md" style="padding-left: 0; padding-right: 0;">
  <q-table
    :rows="collaboratorListData"
    :columns="[]"  <!-- Убираем заголовки -->
    row-key="id"
    flat
    bordered
    hide-header
  >
    <template v-slot:body="props">
      <tr>
        <td class="text-left">
          <q-radio
            v-model="selectedCoworkerId"
            :val="props.row.id"
            color="primary"
            class="q-mb-sm"
          />
          <div><strong>{{ props.row.fullname }}</strong></div>
          <div>{{ props.row.position_name }}</div>
          <div>{{ props.row.department_name }}</div>
        </td>
      </tr>
    </template>
  </q-table>
</div>


