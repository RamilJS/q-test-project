if (ArrayCount(aCities) > 0) {

    bSqlStr.AppendStr("and c.data.value(");

    bSqlStr.AppendStr("'(*/custom_elems/custom_elem[name=''sity'']/value)[1]'");

    bSqlStr.AppendStr(",'varchar(max)')");

    bSqlStr.AppendStr(" in ('");

    bSqlStr.AppendStr(ArrayMerge(aCities, "This", "', '"));

    bSqlStr.AppendStr("')");

    bSqlStr.AppendStr("\r\n");
}
