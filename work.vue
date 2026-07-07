
sCities = String(Trim(Param.cities));
aCities = [];

if (sCities != '') {
    aCities = sCities.split(',');
}

LogEvent("start_adaptation", "Параметр cities = " + sCities);

for (i = 0; i < ArrayCount(aCities); i++) {
    LogEvent(
        "start_adaptation",
        i + " = [" + aCities[i] + "]"
    );
}

for (sCity in aCities) {
    sCity = Trim(sCity);
}
