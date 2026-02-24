// получаем финальную дату
var finalDate = oEventDocTE.custom_elems.ObtainChildByKey("f_final_date").value;

// текущий год (это у тебя уже работает)
var currentYear = Year(Date());

// флаг показа поля
var showYearResult = false;

// проверяем только finalDate
if (finalDate != undefined && finalDate != null && finalDate != "") {

    var finalYear = Year(finalDate);

    // показываем только если год финальной даты НЕ превышает текущий
    if (finalYear <= currentYear) {
        showYearResult = true;
    }
}


// добавляем поле только если нужно
if (showYearResult) {

    RESULT.push({
        "name": "Ожидаемый результат на конец текущего года (критерии успешности):",
        "value": String(oEventDocTE.custom_elems.ObtainChildByKey("f_year_result").value),
        "id": ArrayCount(RESULT)
    });

}
