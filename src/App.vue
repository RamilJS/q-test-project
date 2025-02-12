<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

<script>
import { defineComponent } from 'vue'

export default defineComponent({
  name: 'App'
})
</script>

<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <title>Таблица на CSS Grid</title>
  <style>
    /* Контейнер для нашей "таблицы" */
    .grid-table {
      display: grid;
      grid-template-columns: repeat(6, 1fr); /* 6 равных колонок */
      gap: 1px; /* Разделители между ячейками (как границы) */
      background-color: #ddd; /* Цвет разделителей */
      font-family: Arial, sans-serif;
    }

    /* Стили для каждой ячейки */
    .grid-table div {
      background-color: #fff;
      padding: 10px;
    }

    /* Стили для заголовков */
    .grid-table .header {
      background-color: #f4f4f4;
      font-weight: bold;
      text-transform: uppercase;
    }

    /* Пример эффекта наведения для ячеек */
    .grid-table div:hover {
      background-color: #f1f1f1;
    }
    
    /* Если нужно задать чередование строк, можно группировать ячейки в ряды.
       Здесь показан пример с использованием дополнительной обёртки для строк. */
    .grid-row {
      display: contents; /* Позволяет строке не влиять на сетку, но группировать ячейки */
    }
    
    /* Пример стилизации чётных строк */
    .grid-row:nth-child(even) div {
      background-color: #f9f9f9;
    }
  </style>
</head>
<body>
  <!-- Заголовки таблицы -->
  <div class="grid-table">
    <div class="header">Заголовок 1</div>
    <div class="header">Заголовок 2</div>
    <div class="header">Заголовок 3</div>
    <div class="header">Заголовок 4</div>
    <div class="header">Заголовок 5</div>
    <div class="header">Заголовок 6</div>
    
    <!-- Первая строка данных -->
    <div class="grid-row">
      <div>Ячейка 1</div>
      <div>Ячейка 2</div>
      <div>Ячейка 3</div>
      <div>Ячейка 4</div>
      <div>Ячейка 5</div>
      <div>Ячейка 6</div>
    </div>
    
    <!-- Вторая строка данных -->
    <div class="grid-row">
      <div>Ячейка 1</div>
      <div>Ячейка 2</div>
      <div>Ячейка 3</div>
      <div>Ячейка 4</div>
      <div>Ячейка 5</div>
      <div>Ячейка 6</div>
    </div>
    
    <!-- Добавляйте дополнительные строки по необходимости -->
  </div>
</body>
</html>
