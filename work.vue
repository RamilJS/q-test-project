var finalDateStr = String(oEventDocTE.custom_elems.ObtainChildByKey("f_final_date").value);
var showYearResult = false;

// получаем текущий год из CurDate (это поддерживается)
var currentYearStr = String(CurDate).substring(0, 4);

if (finalDateStr != "") {

    var finalYearStr = finalDateStr.substring(0, 4);

    if (finalYearStr != currentYearStr) {
        showYearResult = true;
    }

}

// всегда показываем финальную дату
RESULT.push({
    "name": "Финальная дата завершения:",
    "value": finalDateStr,
    "id": ArrayCount(RESULT)
});

// показываем year result только если год отличается
if (showYearResult) {

    RESULT.push({
        "name": "Ожидаемый результат на конец текущего года (критерии успешности):",
        "value": String(oEventDocTE.custom_elems.ObtainChildByKey("f_year_result").value),
        "id": ArrayCount(RESULT)
    });

}
