var finalDateStr = String(oEventDocTE.custom_elems.ObtainChildByKey("f_final_date").value);

// получаем текущий год
var currentYear = Int(StrLeft(StrDate(CurDate, false), 4));

var showYearResult = false;

if (finalDateStr != "") {

    var finalYear = Int(StrLeft(finalDateStr, 4));

    if (finalYear != currentYear) {
        showYearResult = true;
    }
}

// Финальная дата
RESULT.push({
    "name": "Финальная дата завершения:",
    "value": finalDateStr,
    "id": ArrayCount(RESULT)
});

// Только если год НЕ равен текущему
if (showYearResult) {

    RESULT.push({
        "name": "Ожидаемый результат на конец текущего года (критерии успешности):",
        "value": String(oEventDocTE.custom_elems.ObtainChildByKey("f_year_result").value),
        "id": ArrayCount(RESULT)
    });

}
