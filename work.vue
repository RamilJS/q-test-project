var finalDateElem = oEventDocTE.custom_elems.ObtainChildByKey("f_final_date");
var yearResultElem = oEventDocTE.custom_elems.ObtainChildByKey("f_year_result");

var finalDate = finalDateElem != undefined ? finalDateElem.value : undefined;

// всегда показываем финальную дату
RESULT.push({
    "name": "Финальная дата завершения:",
    "value": finalDate ? StrDate(finalDate) : "",
    "id": ArrayCount(RESULT)
});


// проверяем нужно ли показывать year_result
if (finalDate != undefined && finalDate != null && finalDate != "") {

    var finalYear = Year(finalDate);
    var currentYear = Year(Date());

    if (finalYear != currentYear) {

        RESULT.push({
            "name": "Ожидаемый результат на конец текущего года:",
            "value": yearResultElem != undefined ? ("" + yearResultElem.value) : "",
            "id": ArrayCount(RESULT)
        });

    }

}
