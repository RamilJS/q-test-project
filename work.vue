var subdivisionId = OptInt(oEventDocTE.custom_elems.ObtainChildByKey("f_vacancy_subdivision_id").value);
var subdivisionName = Trim(String(oEventDocTE.custom_elems.ObtainChildByKey("f_vacancy_subdivision_name").value));

var hierarchyFinal = "";

// -------------------------------------------
// 1. PRIORITY: если subdivisionName есть → используем только его
// -------------------------------------------
if (subdivisionName != "") {
    hierarchyFinal = subdivisionName;
}

// -------------------------------------------
// 2. Если имени нет, но есть ID — ищем по справочникам
// -------------------------------------------
else if (subdivisionId > 0) {

    var subArr = XQuery("for $s in subdivisions where $s/id = " + subdivisionId + " return $s");
    var sub = ArrayOptFirstElem(subArr);

    if (sub != undefined) {

        // --- ищем организацию ---
        var orgName = "";
        var orgId = OptInt(sub.org_id);

        if (orgId > 0) {
            var orgArr = XQuery("for $o in orgs where $o/id = " + orgId + " return $o");
            var org = ArrayOptFirstElem(orgArr);
            if (org != undefined)
                orgName = String(org.name);
        }

        // --- строим иерархию ---
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

        // --- итоговая строка ---
        var segments = [];

        if (orgName != "")
            segments.push(orgName);

        if (hierarchy.length > 0)
            segments.push(hierarchy.join(" → "));

        segments.push(String(sub.name));

        hierarchyFinal = segments.join(" → ");
    }
}

// -------------------------------------------
// 3. Если оба поля пустые → пусто
// -------------------------------------------
// hierarchyFinal уже ""


