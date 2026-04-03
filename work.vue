var arr = XQuery(
    "for $l in test_learnings " +
    "where $l/person_id = " + XQueryLiteral(person_id) +
    " and $l/assessment_id = " + XQueryLiteral(ass_id) +
    " and $l/start_usage_date >= date('" + StrDate(dStartDate,false) + "')" +
    " and $l/start_usage_date <= date('" + StrDate(dFinishDate,false) + "')" +
    " return $l"
);

alert("Array = " + arr);

// Нормализуем результат в массив
var items = [];
if (arr != undefined && arr != null) {
    if (ArrayIsArray(arr)) {
        items = arr;
    } else if (arr.id != undefined) {
        // Одиночный XML-узел
        items = [arr];
    }
}

for (var k = 0; k < ArrayCount(items); k++)
{
    var l = items[k];
    // ... остальной код без изменений
}
