var finalDateStr = String(oEventDocTE.custom_elems.ObtainChildByKey("f_final_date").value);
var currentYear = Int(StrLeft(String(curEnv.nowDate), 4));

if (finalDateStr != "") {

    RESULT.push({
        "name": "Финальная дата завершения:",
        "value": finalDateStr,
        "id": ArrayCount(RESULT)
    });

    if (Int(StrLeft(finalDateStr, 4)) != currentYear) {

        RESULT.push({
            "name": "Ожидаемый результат на конец текущего года (критерии успешности):",
            "value": String(oEventDocTE.custom_elems.ObtainChildByKey("f_year_result").value),
            "id": ArrayCount(RESULT)
        });

    }
}
