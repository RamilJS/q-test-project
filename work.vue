// получаем финальную дату из XML (строка)
var finalDateStr = oEventDocTE.custom_elems.ObtainChildByKey("f_final_date").value;

// текущий год
var currentYear = Year(Date());

// флаг показа поля
var showYearResult = false;

// проверяем, что дата есть и не пустая
if (finalDateStr != undefined && finalDateStr != null && finalDateStr != "") {

    // Берем первые 4 символа строки — это год
    var finalYearStr = String(finalDateStr).substr(0,4);

    // преобразуем в число
    var finalYear = Int(finalYearStr);

    // показываем поле f_year_result только если год финальной даты НЕ совпадает с текущим
    if (finalYear != currentYear) {
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
