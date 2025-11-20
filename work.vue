iPersonID = Request.AuthUserID;
requestTypeID = 7194701844403546882;

var aReqs = XQuery("
    for $r in requests
    where $r/person_id = " + iPersonID + "
      and $r/request_type_id = " + requestTypeID + "
    return $r
");

RESULT = [];

for (oElem in aReqs) {

    doc = OpenDoc(UrlFromDocID(oElem.id));
    root = doc.TopElem;

    vacancyName = root.custom_elems.ObtainChildByKey("f_vacancy_name").value;
    duties      = root.custom_elems.ObtainChildByKey("f_candidate_function").value;

    RESULT.push({
        id: String(Int(oElem.id)),
        disp: vacancyName,
        d0: vacancyName,
        d1: String(root.create_date),
        d2: duties
    });
}


