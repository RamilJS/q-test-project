// вычисляем флаг видимости yearResult
var yearResultVisibleFlag = "0";

var finalDateStr = getFormField("finalDate", "");

if (finalDateStr != "")
{
    var currentYear = Year(Date());
    var finalYear = Int(String(finalDateStr).substr(0,4));

    if (finalYear > currentYear)
        yearResultVisibleFlag = "1";
}

{
    name: "yearResultVisibleFlag",
    type: "string",
    value: yearResultVisibleFlag,
    visibility: true,
    page: "page_one"
},
