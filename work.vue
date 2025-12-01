RESULT = [];

iEventID = OptInt(request_id);
oEventDocTE = tools.open_doc(iEventID).TopElem;

sResType = result_type;

// ---------------------------------------------------------
// 1. Поиск руководителя
// ---------------------------------------------------------
iPersonId = oEventDocTE.person_id;

oBossObj = {
    id: undefined,
    fullname: undefined
};

oPersonDocTE = tools.open_doc(iPersonId).TopElem;
oBoss = ArrayOptFind(oPersonDocTE.func_managers, "This.is_native");

if (oBoss != undefined) {
    oBossObj.id = oBoss.person_id;
    oBossObj.fullname = oBoss.person_id.ForeignElem.fullname;
}

// ---------------------------------------------------------
// 2. Поиск подразделения, организации и построение иерархии
// ---------------------------------------------------------

var subdivisionId = OptInt(oEventDocTE.custom_elems.ObtainChildByKey("f_vacancy_subdivision_id").value);

var oOrgObj = { id: undefined, name: undefined };
var oSubObj = { id: subdivisionId, name: undefined, full_path: "" };

if (subdivisionId > 0) {

    // --- ищем конечное подразделение ---
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
        // ЦИКЛ: собираем всю иерархию parent_object_id
        // ---------------------------------------------------------

        var hierarchy = [];
        var currentParentId = OptInt(sub.parent_object_id);

        while (currentParentId > 0) {
            var pArr = XQuery("for $p in subdivisions where $p/id = " + currentParentId + " return $p");
            var p = ArrayOptFirstElem(pArr);

            if (p == undefined)
                break;

            hierarchy.push(String(p.name));

            // переходим выше
            currentParentId = OptInt(p.parent_object_id);
        }

        // теперь иерархию надо перевернуть — чтобы шло сверху вниз
        hierarchy = hierarchy.reverse();

        // сохраняем в объект
        oSubObj.full_path = hierarchy.join(" → ");
    }
}


// ---------------------------------------------------------
// 3. Формируем итоговую строку из орг + иерархия + подразделение
// ---------------------------------------------------------

var hierarchyNames = [];

if (oOrgObj.name != undefined)
    hierarchyNames.push(oOrgObj.name);

if (oSubObj.full_path != "")
    hierarchyNames.push(oSubObj.full_path);

if (oSubObj.name != undefined)
    hierarchyNames.push(oSubObj.name);

var hierarchyFinal = hierarchyNames.join(" → ");


// ---------------------------------------------------------
// 4. ФОРМИРУЕМ РЕЗУЛЬТАТ (fields)
// ---------------------------------------------------------

switch (sResType) {

    case "fields": {

        RESULT.push({
            "name": "Дата подачи",
            "value": StrDate(oEventDocTE.create_date, false, false),
            "id": ArrayCount(RESULT)
        });

        RESULT.push({
            "name": "Наименование",
            "value": String(oEventDocTE.custom_elems.ObtainChildByKey("f_vacancy_name").value),
            "id": ArrayCount(RESULT)
        });

        RESULT.push({
            "name": "Штатное расписание",
            "value": hierarchyFinal,
            "id": ArrayCount(RESULT)
        });

        RESULT.push({
            "name": "Текущий статус",
            "value": String(oEventDocTE.workflow_state_name),
            "id": ArrayCount(RESULT)
        });

        RESULT.push({
            "name": "ФИО руководителя",
            "value": String(oBossObj.fullname),
            "id": ArrayCount(RESULT)
        });

        break;
    }

    default:
        break;
}
