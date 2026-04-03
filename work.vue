var arr = XQuery(
    "for $l in test_learnings " +
    "where $l/person_id = " + XQueryLiteral(person_id) +
    " and $l/assessment_id = " + XQueryLiteral(ass_id) +
    " and $l/start_usage_date >= date('" + StrDate(dStartDate,false) + "')" +
    " and $l/start_usage_date <= date('" + StrDate(dFinishDate,false) + "')" +
    " return $l"
);

alert("Array = " + arr);

// --- Нормализация результата в массив ---
var items = [];
if (arr != undefined && arr != null) {
    if (ArrayCount(arr) > 0 && arr.id === undefined) {
        items = arr;
    } else if (arr.id !== undefined) {
        items = [arr];
    }
}

for (var k = 0; k < ArrayCount(items); k++) {
    var l = items[k];
    // ... всё остальное без изменений
}
