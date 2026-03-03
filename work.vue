var subprojectValueRaw = getFormField("subprojectValue", "");

if (subprojectValueRaw == "")
{
    aErrorArray.push("Выберите вес подпроекта.");
}
else
{
    // приводим к строке, убираем пробелы и %
    var cleanedValue = Trim(String(subprojectValueRaw));
    cleanedValue = Replace(cleanedValue, "%", "");

    // пробуем преобразовать в число
    var weight = Int(cleanedValue);

    // если не число или меньше 5
    if (weight < 5)
        aErrorArray.push("Вес подпроекта не может быть меньше 5%.");
}
