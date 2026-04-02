var oSelect_collaborator_ids = getFormField('select_collaborator_ids', "");

if (oSelect_collaborator_ids != undefined && oSelect_collaborator_ids != "")
{
    if (TypeName(oSelect_collaborator_ids) == "string")
        oSelect_collaborator_ids = StrSplit(oSelect_collaborator_ids, ";");
    else
        oSelect_collaborator_ids = [oSelect_collaborator_ids];
}
else
{
    oSelect_collaborator_ids = [];
}


var oSelect_assessment_ids = getFormField('select_assessment_ids', "");

if (oSelect_assessment_ids != undefined && oSelect_assessment_ids != "")
{
    if (TypeName(oSelect_assessment_ids) == "string")
        oSelect_assessment_ids = StrSplit(oSelect_assessment_ids, ";");
    else
        oSelect_assessment_ids = [oSelect_assessment_ids];
}
else
{
    oSelect_assessment_ids = [];
}
