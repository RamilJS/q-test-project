// --- Сохраняем спонсоров ---
var sponsorStr = getFormField("sponsorName", "");

// sponsorStr приходит как:
// "123456;789012;345678"

if (sponsorStr != "")
{
    var sponsorIds = String(sponsorStr).split(";");

    for (var i = 0; i < ArrayCount(sponsorIds); i++)
    {
        var sponsorId = sponsorIds[i];

        if (sponsorId == "")
            continue;

        // открываем collaborator
        var sponsorDoc = tools.open_doc(sponsorId);
        if (sponsorDoc == undefined)
            continue;

        var sponsorTE = sponsorDoc.TopElem;

        // создаём новый узел person
        var newPerson = oReqDocTE.persons.AddChild();

        newPerson.person_id = sponsorTE.id;
        newPerson.person_fullname = sponsorTE.fullname;

        newPerson.person_position_id = sponsorTE.position_id;
        newPerson.person_position_name = sponsorTE.position_name;
        newPerson.person_position_code = sponsorTE.position_code;

        newPerson.person_org_id = sponsorTE.org_id;
        newPerson.person_org_name = sponsorTE.org_name;
        newPerson.person_org_code = sponsorTE.org_code;

        newPerson.person_subdivision_id = sponsorTE.position_parent_id;
        newPerson.person_subdivision_name = sponsorTE.position_parent_name;
        newPerson.person_subdivision_code = sponsorTE.position_parent_code;

        newPerson.person_code = sponsorTE.code;
    }
}
