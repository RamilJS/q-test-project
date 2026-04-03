var arr = XQuery(
    "for $l in test_learnings " +
    "where $l/person_id = " + XQueryLiteral(person_id) +
    " and $l/assessment_id = " + XQueryLiteral(ass_id) +
    " and $l/start_usage_date >= date('" + StrDate(dStartDate,false) + "')" +
    " and $l/start_usage_date <= date('" + StrDate(dFinishDate,false) + "')" +
    " return $l"
);

alert("Array = " + arr);

// Нормализуем результат в массив (без использования .length)
var items = [];
if (arr != undefined && arr != null && ArrayCount(arr) > 0) {
    // Если найден ровно один элемент и у него есть поле id (значит, это узел, а не массив)
    if (ArrayCount(arr) == 1 && arr.id !== undefined) {
        items = [arr];
    } else {
        // Иначе считаем, что arr уже массив (или коллекция, поддерживающая индексацию)
        items = arr;
    }
}

// Перебираем все найденные попытки
for (var k = 0; k < ArrayCount(items); k++)
{
    var l = items[k];
    // ... остальной код без изменений
}
