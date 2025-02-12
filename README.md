# Quasar App (q-test-project)

 quasar app for test

## Install the dependencies
```bash
yarn
# or
npm install
```

### Start the app in development mode (hot-code reloading, error reporting, etc.)
```bash
quasar dev
```


### Lint the files
```bash
yarn lint
# or
npm run lint
```



### Build the app for production
```bash
quasar build
```

### Customize the configuration
See [Configuring quasar.config.js](https://v2.quasar.dev/quasar-cli-vite/quasar-config-js).

<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <title>Таблица с шестью колонками</title>
  <style>
    /* Стили для таблицы */
    table {
      width: 100%; /* Таблица занимает всю ширину контейнера */
      border-collapse: collapse; /* Убираем промежутки между границами ячеек */
      font-family: Arial, sans-serif;
    }

    th, td {
      border: 1px solid #ddd; /* Граница для ячеек */
      padding: 10px; /* Отступы внутри ячеек */
      text-align: left; /* Выравнивание текста по левому краю */
    }

    th {
      background-color: #f4f4f4; /* Цвет фона для заголовков */
      font-weight: bold;
      text-transform: uppercase; /* Приведение текста к верхнему регистру */
    }

    /* Чередование цвета строк для удобства чтения */
    tr:nth-child(even) {
      background-color: #f9f9f9;
    }

    /* Подсветка строки при наведении */
    tr:hover {
      background-color: #f1f1f1;
    }

    /* Адаптивные стили для экранов с шириной до 600px */
    @media (max-width: 600px) {
      table {
        font-size: 14px;
      }
      
      th, td {
        padding: 8px;
      }
    }
  </style>
</head>
<body>
  <table>
    <thead>
      <tr>
        <th>Заголовок 1</th>
        <th>Заголовок 2</th>
        <th>Заголовок 3</th>
        <th>Заголовок 4</th>
        <th>Заголовок 5</th>
        <th>Заголовок 6</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>Ячейка 1</td>
        <td>Ячейка 2</td>
        <td>Ячейка 3</td>
        <td>Ячейка 4</td>
        <td>Ячейка 5</td>
        <td>Ячейка 6</td>
      </tr>
      <tr>
        <td>Ячейка 1</td>
        <td>Ячейка 2</td>
        <td>Ячейка 3</td>
        <td>Ячейка 4</td>
        <td>Ячейка 5</td>
        <td>Ячейка 6</td>
      </tr>
      <!-- Здесь можно добавлять дополнительные строки по необходимости -->
    </tbody>
  </table>
</body>
</html>
