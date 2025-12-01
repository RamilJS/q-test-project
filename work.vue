var subdivisionId = OptInt(oEventDocTE.custom_elems.ObtainChildByKey("f_vacancy_subdivision_id").value);
var subdivisionNameFromField = Trim(String(oEventDocTE.custom_elems.ObtainChildByKey("f_vacancy_subdivision_name").value));

var oOrgObj = { id: undefined, name: undefined };
var oSubObj = { id: subdivisionId, name: subdivisionNameFromField, full_path: "" };

// ---------------------------------------------------------
// ЕСЛИ subdivisionId пустой → выводим только subdivisionNameFromField
// ---------------------------------------------------------
if (subdivisionId == 0 || subdivisionId == undefined) {

    // просто используем текст из f_vacancy_subdivision_name
    hierarchyFinal = subdivisionNameFromField;

} else {

    // ---------------------------------------------------------
    // Иначе выполняем обычную логику поиска подразделения
    // ---------------------------------------------------------

    var subArr = XQuery("for $s in subdivisions where $s/id = " + subdivisionId + " return $s");
    var sub = ArrayOptFirstElem(subArr);

    if (sub != undefined) {

        oSubObj.name = sub.name;

        // --- ищем организацию ---
        var orgId = OptInt(sub.org_id);
        if (orgId > 0) {
            var orgArr = XQuery("for $o in orgs where $o/id = " + orgId + " return $o");
            var org = ArrayOptFirstElem(orgArr);

            if (org != undefined) {
                oOrgObj.id = org.id;
                oOrgObj.name = String(org.name);
            }
        }

        // ---------------------------------------------------------
        // Иерархия parent_object_id
        // ---------------------------------------------------------
        var hierarchy = [];
        var currentParentId = OptInt(sub.parent_object_id);

        while (currentParentId > 0) {
            var pArr = XQuery("for $p in subdivisions where $p/id = " + currentParentId + " return $p");
            var p = ArrayOptFirstElem(pArr);

            if (p == undefined)
                break;

            hierarchy.push(String(p.name));
            currentParentId = OptInt(p.parent_object_id);
        }

        hierarchy = hierarchy.reverse();
        oSubObj.full_path = hierarchy.join(" → ");

        // итоговая сборка
        var hierarchyNames = [];

        if (oOrgObj.name != undefined)
            hierarchyNames.push(oOrgObj.name);

        if (oSubObj.full_path != "")
            hierarchyNames.push(oSubObj.full_path);

        hierarchyNames.push(oSubObj.name);

        hierarchyFinal = hierarchyNames.join(" → ");
    }
}
