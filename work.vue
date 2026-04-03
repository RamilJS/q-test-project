var arr = XQuery(
    "for $l in test_learnings " +
    "where $l/person_id = " + XQueryLiteral(person_id) +
    " and $l/assessment_id = " + XQueryLiteral(ass_id) +
    " and $l/start_usage_date >= date('" + StrDate(dStartDate,false) + "')" +
    " and $l/start_usage_date <= date('" + StrDate(dFinishDate,false) + "')" +
    " return $l"
);

alert("Array = " + arr);

// --- Нормализация результата в массив (без ArrayIsArray) ---
var items = [];
if (arr != undefined && arr != null) {
    // Проверка: если это массив (имеет свойство length и не имеет поля id, характерного для узла)
    if (typeof arr.length === "number" && arr.id === undefined) {
        items = arr;
    } else if (arr.id !== undefined) {
        // Одиночный XML-узел
        items = [arr];
    } else {
        // Пустой результат или иной случай
        items = [];
    }
}

// --- Цикл по найденным попыткам ---
for (var k = 0; k < items.length; k++)
{
    var l = items[k];
    // ... остальной код без изменений
}
