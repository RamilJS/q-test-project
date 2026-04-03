var arr = XQuery(
    "for $l in (active_test_learnings, test_learnings) " +
    "where $l/person_id = " + person_id +
    " and $l/assessment_id = " + ass_id +
    " and $l/start_usage_date >= date('" + StrDate(dStartDate,false) + "')" +
    " and $l/start_usage_date <= date('" + StrDate(dFinishDate,false) + "')" +
    " return $l"
);
