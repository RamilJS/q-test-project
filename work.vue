
LogEvent("start_adaptation", "Сотрудник: " + collaboratorDocTE.fullname);
LogEvent("start_adaptation", "Город из карточки = [" + sCollabCity + "]");

if (sCities != '') {
    aCities = sCities.split(',');
}

LogEvent("start_adaptation", "Параметр cities = " + sCities);

for (city in aCities) {
    LogEvent("start_adaptation", "Элемент массива = [" + city + "]");
}

for (sCity in aCities) {
    sCity = Trim(sCity);
}
