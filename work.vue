var subdivisionId = OptInt(oEventDocTE.custom_elems.ObtainChildByKey("f_vacancy_subdivision_id").value);

var oOrgObj = {
    org_id: undefined,
    org_name: undefined,
    sub_id: subdivisionId,
    sub_name: undefined
};

if (subdivisionId > 0) {

    // 1. Ищем подразделение по ID
    var subArr = XQuery(
        "for $s in subdivisions " +
        "where $s/id = " + subdivisionId + " " +
        "return $s"
    );

    var sub = ArrayOptFirstElem(subArr);

    if (sub != undefined) {

        // ✔ сохраняем название подразделения
        oOrgObj.sub_name = String(sub.name);

        var orgId = OptInt(sub.org_id);

        // 2. Ищем организацию, к которой относится это подразделение
        if (orgId > 0) {

            var orgArr = XQuery(
                "for $o in organizations " +
                "where $o/id = " + orgId + " " +
                "return $o"
            );

            var org = ArrayOptFirstElem(orgArr);

            if (org != undefined) {

                // ✔ сохраняем организацию
                oOrgObj.org_id = org.id;
                oOrgObj.org_name = String(org.name);
            }
        }
    }
}
