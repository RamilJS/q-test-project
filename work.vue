var finalDate = oEventDocTE.custom_elems.ObtainChildByKey("f_final_date").value;
var showYearResult = false;

if (finalDate != undefined && finalDate != null && finalDate != "") {

    var finalYear = Year(finalDate);
    var currentYear = Year(Date());

    if (finalYear != currentYear) {
        showYearResult = true;
    }

}

// Финальная дата
RESULT.push({
    "name": "Финальная дата завершения:",
    "value": StrDate(finalDate),
    "id": ArrayCount(RESULT)
});

// Показываем только если год НЕ текущий
if (showYearResult) {

    RESULT.push({
        "name": "Ожидаемый результат на конец текущего года:",
        "value": "" + oEventDocTE.custom_elems.ObtainChildByKey("f_year_result").value,
        "id": ArrayCount(RESULT)
    });

}
