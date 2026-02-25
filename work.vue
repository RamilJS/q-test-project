// получаем финальную дату
var finalDateStr = getFormField("finalDate", "");
var currentYear = Year(Date());
var showYearResult = false;

if (finalDateStr != undefined && finalDateStr != null && finalDateStr != "") {

    // finalDate приходит как 2027-02-27T00:00:00+00:00
    var finalYearStr = String(finalDateStr).substr(0,4);
    var finalYear = Int(finalYearStr);

    // показываем только если год больше текущего
    if (finalYear > currentYear) {
        showYearResult = true;
    }
}

// поле "Ожидаемый результат на финальную дату"
oForm.form_fields.push({
    name: "result",
    label: "Ожидаемый результат на финальную дату завершения (критерии успешности)",
    type: "text",
    richtext: true,
    value: getFormField("result", ""),
    mandatory: false
});

// добавляем yearResult ТОЛЬКО если finalDate > текущего года
if (showYearResult) {

    oForm.form_fields.push({
        name: "yearResult",
        label: "Ожидаемый результат на конец текущего года (критерии успешности)",
        type: "text",
        richtext: true,
        value: getFormField("yearResult", ""),
        mandatory: false
    });

}
