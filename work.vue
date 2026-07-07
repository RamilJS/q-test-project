
sCollabCity = collaboratorDocTE.custom_elems.ObtainChildByKey('sity').value;

findCity = ArrayOptFind(aCities, "This == sCollabCity");

LogEvent(
    "start_adaptation",
    "Проверка города [" + sCollabCity + "] -> " +
    (findCity == undefined ? "НЕ НАЙДЕН" : "НАЙДЕН [" + findCity + "]")
);

if (ArrayOptFind(aCities, "This == sCollabCity") == undefined && ArrayCount(aCities) > 0 ) {
