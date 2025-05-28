function pluralizeMeetings(count: number): string {
  const mod10 = count % 10;
  const mod100 = count % 100;

  if (mod10 === 1 && mod100 !== 11) return `${count} встреча`;
  if (mod10 >= 2 && mod10 <= 4 && !(mod100 >= 12 && mod100 <= 14)) return `${count} встречи`;
  return `${count} встреч`;

  }

<template>
  <div class="q-mt-sm text-caption text-grey-8">
    {{ pluralizeMeetings(meetingCount) }}
  </div>
</template>

<script setup>
const meetingCount = ref(5) // Пример

function pluralizeMeetings(count) {
  const mod10 = count % 10;
  const mod100 = count % 100;

  if (mod10 === 1 && mod100 !== 11) return `${count} встреча`;
  if (mod10 >= 2 && mod10 <= 4 && !(mod100 >= 12 && mod100 <= 14)) return `${count} встречи`;
  return `${count} встреч`;
}
</script>
