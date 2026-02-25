var sponsorStr = getFormField("sponsorName", "");

// очищаем
if (oReqDocTE.persons != undefined)
{
    for (var i = oReqDocTE.persons.ChildNum - 1; i >= 0; i--)
    {
        oReqDocTE.persons.RemoveChild(i);
    }
}

// добавляем новых
if (sponsorStr != "")
{
    var sponsorIds = String(sponsorStr).split(";");

    for (var i = 0; i < sponsorIds.length; i++)
    {
        var sponsorId = sponsorIds[i];

        if (sponsorId == "")
            continue;

        var sponsorDoc = tools.open_doc(sponsorId);
        if (sponsorDoc == undefined)
            continue;

        var sponsorTE = sponsorDoc.TopElem;

        var newPerson = oReqDocTE.persons.AddChild();

        newPerson.person_id = sponsorTE.id;
        newPerson.person_fullname = sponsorTE.fullname;
        newPerson.person_position_id = sponsorTE.position_id;
        newPerson.person_position_name = sponsorTE.position_name;
        newPerson.person_org_id = sponsorTE.org_id;
        newPerson.person_org_name = sponsorTE.org_name;
    }
}
