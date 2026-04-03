var person_id = "0x" + StrHex(OptInt(raw_person_id));
var ass_id = "0x" + StrHex(OptInt(raw_ass_id));

var arr = XQuery(
    "for $l in (active_test_learnings, test_learnings) " +
    "where $l/person_id = " + person_id +
    " and $l/assessment_id = " + ass_id +
    " and $l/start_usage_date >= date('" + StrDate(dStartDate,false) + "')" +
    " and $l/start_usage_date <= date('" + StrDate(dFinishDate,false) + "')" +
    " return $l"
);

alert("person_id HEX = " + person_id);
alert("ass_id HEX = " + ass_id);
