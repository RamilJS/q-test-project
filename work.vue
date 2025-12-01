var subdivisionId = OptInt(oEventDocTE.custom_elems.ObtainChildByKey("f_vacancy_subdivision_id").value);

var oOrgObj = {
    id: undefined,
    name: undefined
};

if (subdivisionId > 0) {

    // 1. Ищем подразделение
    var subArr = XQuery(
        "for $s in subdivisions " +
        "where $s/id = " + subdivisionId + " " +
        "return $s"
    );

    // subArr — это массив, нужно брать subArr[0]
    var sub = ArrayOptFirstElem(subArr);

    if (sub != undefined) {

        var orgId = OptInt(sub.org_id);

        // 2. Ищем организацию
        if (orgId > 0) {

            var orgArr = XQuery(
                "for $o in organizations " +
                "where $o/id = " + orgId + " " +
                "return $o"
            );

            var org = ArrayOptFirstElem(orgArr);

            if (org != undefined) {
                oOrgObj.id = org.id;
                oOrgObj.name = String(org.name);
            }
        }
    }
}
