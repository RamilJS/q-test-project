function normalizeToArray(val)
{
    if (val == undefined || val == "")
        return [];

    // если уже массив
    if (val instanceof Array)
        return val;

    // если строка с разделителем
    if (StrContains(val, ";"))
        return StrSplit(val, ";");

    // если одиночное значение
    return [val];
}

var oSelect_collaborator_ids = normalizeToArray(getFormField('select_collaborator_ids', ""));
var oSelect_assessment_ids = normalizeToArray(getFormField('select_assessment_ids', ""));
