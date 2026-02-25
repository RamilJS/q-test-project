case "sponsorName":

    var sponsorStr = "";

    if (oReqDocTE.persons != undefined)
    {
        for (var j = 0; j < oReqDocTE.persons.ChildNum; j++)
        {
            var person = oReqDocTE.persons[j];

            if (person.person_id != undefined)
            {
                if (sponsorStr != "")
                    sponsorStr += ";";

                sponsorStr += String(person.person_id);
            }
        }
    }

    fField.value = sponsorStr;

    break;


case "sponsorName":

    var sponsorStr = "";

    if (oReqDocTE.persons != undefined)
    {
        for (var j = 0; j < oReqDocTE.persons.length; j++)
        {
            var person = oReqDocTE.persons[j];

            if (sponsorStr != "")
                sponsorStr += ";";

            sponsorStr += String(person.person_id);
        }
    }

    fField.value = sponsorStr;

    break;
