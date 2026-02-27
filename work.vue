{
    name: "yearResultVisibleFlag",
    type: "string",
    value: (function () {
        var finalDateStr = getFormField("finalDate", "");
        if (finalDateStr == "")
            return "0";

        var currentYear = Year(Date());
        var finalYear = Int(String(finalDateStr).substr(0,4));

        return (finalYear > currentYear ? "1" : "0");
    })(),
    visibility: false,
    page: "page_one"
},

{
    name: "yearResult",
    label: "Ожидаемый результат на конец текущего года (критерии успешности)",
    type: "text",
    richtext: true,
    value: getFormField("yearResult", ""),
    mandatory: false, // важно: mandatory делаем через проверку, не тут
    visibility: [
        { parent: "yearResultVisibleFlag", value: "1" }
    ],
    page: "page_one"
},
