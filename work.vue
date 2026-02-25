// формируем строку спонсоров
var sponsorsDisplay = "";

if (oEventDocTE.persons != undefined && ArrayCount(oEventDocTE.persons) > 0)
{
    var sponsorNames = [];

    for (var i = 0; i < ArrayCount(oEventDocTE.persons); i++)
    {
        var personElem = oEventDocTE.persons[i];

        if (personElem.person_fullname != undefined && personElem.person_fullname != "")
        {
            sponsorNames.push(String(personElem.person_fullname));
        }
    }

    sponsorsDisplay = sponsorNames.join("; ");
}

RESULT.push({
    "name": "Спонсоры",
    "value": sponsorsDisplay,
    "id": ArrayCount(RESULT)
});
