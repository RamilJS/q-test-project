function normalizeIds(val) {
    if (val == undefined || val == null || val == "")
        return [];

    // если вдруг прилетел объект
    if (TypeName(val) == "object") {
        if (val.id != undefined)
            return [Str(val.id)];
    }

    val = String(val);

    if (val == "")
        return [];

    return val.split(";");
}

var oSelect_collaborator_ids = normalizeIds(getFormField('select_collaborator_ids', ""));
var oSelect_assessment_ids = normalizeIds(getFormField('select_assessment_ids', ""));
