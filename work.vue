<q-date v-model="selectedDate">
  <template #day="scope">
    <div style="background:red;color:white">
      {{ scope.day }}
    </div>
  </template>
</q-date>
