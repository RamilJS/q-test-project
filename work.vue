// получаем финальную дату из XML
var finalDateStr = oEventDocTE.custom_elems.ObtainChildByKey("f_final_date").value;

// текущий год
var currentYear = Year(Date());

// флаг показа поля
var showYearResult = false;

// проверяем, что дата есть и не пустая
if (finalDateStr != undefined && finalDateStr != null && finalDateStr != "") {

    // преобразуем строку ISO в объект даты Websoft
    var finalDateObj = DateNewTime(finalDateStr);

    // получаем год финальной даты
    var finalYear = Year(finalDateObj);

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
