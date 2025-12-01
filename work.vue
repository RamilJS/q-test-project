// ID подразделения из заявки
var subdivisionId = OptInt(oEventDocTE.custom_elems.ObtainChildByKey("f_vacancy_subdivision_id").value);

// Объект результата
var oOrgObj = new Object();
oOrgObj.id = undefined;
oOrgObj.name = undefined;

if (subdivisionId > 0) {

    // 1. Ищем подразделение
    var subArr = XQuery(
        "for $s in subdivisions " +
        "where $s/id = " + subdivisionId + " " +
        "return $s"
    );

    if (ArrayCount(subArr) > 0) {

        var sub = subArr[0];

        var orgId = OptInt(sub.org_id);

        if (orgId > 0) {

            // 2. Ищем организацию по org_id
            var orgArr = XQuery(
                "for $o in organizations " +
                "where $o/id = " + orgId + " " +
                "return $o"
            );

            if (ArrayCount(orgArr) > 0) {
                oOrgObj.id = orgArr[0].id;
                oOrgObj.name = String(orgArr[0].name);
            }
        }
    }
}
