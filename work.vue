// ------------------------------------------------------------------
// Собираем штатное подразделение: priority -> f_vacancy_subdivision_name
// иначе — идём по subdivision_id вверх по parent_object_id и берём org.name
// ------------------------------------------------------------------

var subdivisionId = OptInt(oEventDocTE.custom_elems.ObtainChildByKey("f_vacancy_subdivision_id").value);
var subdivisionNameFromField = Trim(String(oEventDocTE.custom_elems.ObtainChildByKey("f_vacancy_subdivision_name").value));

var hierarchyFinal = ""; // итоговая строка для вывода
var orgNameResolved = "";
var subNameResolved = "";

// 1) Приоритет — явное имя из поля f_vacancy_subdivision_name
if (subdivisionNameFromField != "") {
    hierarchyFinal = subdivisionNameFromField;
}
else if (subdivisionId > 0) {
    // 2) Ищем саму запись subdivision
    var subdivisionArray = XQuery("for $s in subdivisions where $s/id = " + subdivisionId + " return $s");
    var subdivisionRec = ArrayOptFirstElem(subdivisionArray);

    if (subdivisionRec != undefined) {
        subNameResolved = String(subdivisionRec.name);

        // 2.a) ищем org, если он указан в subdivision
        var orgIdCandidate = OptInt(subdivisionRec.org_id);
        if (orgIdCandidate > 0) {
            var orgArray = XQuery("for $o in orgs where $o/id = " + orgIdCandidate + " return $o");
            var orgRec = ArrayOptFirstElem(orgArray);
            if (orgRec != undefined) {
                orgNameResolved = String(orgRec.name);
            }
        }

        // 2.b) собираем всех родителей subdivision по цепочке parent_object_id
        var parents = []; // накопим имена родителей в порядке bottom->top (будем разворачивать вручную)
        var currentParentId = OptInt(subdivisionRec.parent_object_id);

        while (currentParentId > 0) {
            var parentArr = XQuery("for $p in subdivisions where $p/id = " + currentParentId + " return $p");
            var parentRec = ArrayOptFirstElem(parentArr);
            if (parentRec == undefined) break;

            parents.push(String(parentRec.name));
            // переход к следующему уровню вверх
            currentParentId = OptInt(parentRec.parent_object_id);
        }

        // 2.c) формируем итог в порядке: org (если есть) -> (верхняя цепочка родителей от ближай к корню) -> ... -> непосредственное подразделение
        var segments = [];

        if (orgNameResolved != "") {
            segments.push(orgNameResolved);
        }

        // parents содержит имена от ближай родителя до самого верхнего — нужно вывести от верхнего к ближайшему
        for (var idx = parents.length - 1; idx >= 0; idx--) {
            segments.push(parents[idx]);
        }

        // добавляем конечное (текущее) подразделение
        if (subNameResolved != "") {
            segments.push(subNameResolved);
        }

        // собираем строку (разделитель — стрелка, можешь заменить на " / " или другой)
        hierarchyFinal = segments.join(" → ");
    }
}

// если и name и id отсутствуют — hierarchyFinal остаётся пустой строкой

// --- пример использования: кладём в результат ---
RESULT = RESULT || [];
RESULT.push({
    "name": "Штатное расписание",
    "value": hierarchyFinal,
    "id": ArrayCount(RESULT)
});
