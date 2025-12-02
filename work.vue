RESULT = [];

iEventID = OptInt(request_id);
oEventDocTE = tools.open_doc(iEventID).TopElem;
sResType = result_type;

// ----------------------------
// Поиск руководителя
// ----------------------------
var iPersonId = oEventDocTE.person_id;
var oBossObj = { id: undefined, fullname: undefined };

var oPersonDocTE = tools.open_doc(iPersonId).TopElem;
var oBoss = ArrayOptFind(oPersonDocTE.func_managers, "This.is_native");

if (oBoss != undefined) {
    oBossObj.id = oBoss.person_id;
    oBossObj.fullname = oBoss.person_id.ForeignElem.fullname;
}

// -------------------------------------------------
//  ШТАТНОЕ РАСПИСАНИЕ — ОСНОВНАЯ НОВАЯ ЛОГИКА
// -------------------------------------------------

var subdivisionId = OptInt(oEventDocTE.custom_elems.ObtainChildByKey("f_vacancy_subdivision_id").value);
var subdivisionNameFromField = Trim(String(oEventDocTE.custom_elems.ObtainChildByKey("f_vacancy_subdivision_name").value));

var hierarchyFinal = "";

// ----------------------------
// LOGIC:
// 1) Если subdivisionId есть → ИГНОРИРУЕМ f_vacancy_subdivision_name и строим полный путь
// 2) Если subdivisionId нет → выводим subdivisionNameFromField
// ----------------------------

// -------- case 1: есть subdivision_id ----------
if (subdivisionId > 0) {

    var subdivisionArray = XQuery("for $s in subdivisions where $s/id = " + subdivisionId + " return $s");
    var subdivisionRec = ArrayOptFirstElem(subdivisionArray);

    if (subdivisionRec != undefined) {

        var segments = [];

        // --- Организация ---
        var orgIdCandidate = OptInt(subdivisionRec.org_id);
        if (orgIdCandidate > 0) {
            var orgArr = XQuery("for $o in orgs where $o/id = " + orgIdCandidate + " return $o");
            var orgRec = ArrayOptFirstElem(orgArr);
            if (orgRec != undefined)
                segments.push(String(orgRec.name));
        }

        // --- Родители вверх по иерархии ---
        var parents = [];
        var currentParentId = OptInt(subdivisionRec.parent_object_id);

        while (currentParentId > 0) {
            var parentArr = XQuery("for $p in subdivisions where $p/id = " + currentParentId + " return $p");
            var parentRec = ArrayOptFirstElem(parentArr);

            if (parentRec == undefined)
                break;

            parents.push(String(parentRec.name));
            currentParentId = OptInt(parentRec.parent_object_id);
        }

        // Родители идут от верхнего → к нижнему
        for (var idx = parents.length - 1; idx >= 0; idx--)
            segments.push(parents[idx]);

        // --- Текущее подразделение ---
        segments.push(String(subdivisionRec.name));

        // Собираем строку
        hierarchyFinal = segments.join(" / ");
    }
}
// -------- case 2: subdivision_id нет ----------
else if (subdivisionNameFromField != "") {
    hierarchyFinal = subdivisionNameFromField;
}
// оба поля пустые
else {
    hierarchyFinal = "";
}

// --------------------------------------------
// SWITCH — ФОРМИРОВАНИЕ РЕЗУЛЬТАТОВ
// --------------------------------------------
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

    default: {
        break;
    }
}

