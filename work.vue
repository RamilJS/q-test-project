function IsEventWithoutDates(requestDoc)
{
    result = false;
    try {
        EnableLog("ramil-dates-debug", true);

        oEducationMethodTE = tools.open_doc(requestDoc.TopElem.object_id).TopElem;
        alert("oEducationMethodTE.id = " + oEducationMethodTE.id);
        alert("oEducationMethodTE.name = " + oEducationMethodTE.name);

        valueDates = oEducationMethodTE.custom_elems.ObtainChildByKey("f_vtbl_dates").value;
        alert("valueDates = " + valueDates);

        aDates = ParseJson(valueDates != "" ? valueDates : "[]");
        alert("aDates count = " + ArrayCount(aDates));

        firstDate = ArrayOptFirstElem(aDates);
        alert("firstDate = " + tools.object_to_text(firstDate, "json"));

        if (firstDate != undefined) {
            startDate = Date(firstDate.start_date);
            alert("startDate = " + startDate);
            alert("Day = " + Day(startDate) + ", Month = " + Month(startDate));

            if (Day(startDate) == 1 && Month(startDate) == 1)
                result = true;
        }

        alert("IsEventWithoutDates result = " + result);
    }
    catch(_e) {
        alert("IsEventWithoutDates ERROR: " + _e);
        result = false;
    }
    return result;
}
