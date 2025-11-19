iPersonID = curUserID;

bSqlStr = new Binary();

bSqlStr.AppendStr("select \r\n");
bSqlStr.AppendStr("    r.id, \r\n");
bSqlStr.AppendStr("    r.person_id, \r\n");
bSqlStr.AppendStr("    r.create_date, \r\n");
bSqlStr.AppendStr("    ce_name.value as f_vacancy_name, \r\n");
bSqlStr.AppendStr("    ce_func.value as f_candidate_function \r\n");
bSqlStr.AppendStr("from requests r \r\n");

bSqlStr.AppendStr("left join request_custom_elems ce_name \r\n");
bSqlStr.AppendStr("    on ce_name.request_id = r.id \r\n");
bSqlStr.AppendStr("    and ce_name.name = 'f_vacancy_name' \r\n");

bSqlStr.AppendStr("left join request_custom_elems ce_func \r\n");
bSqlStr.AppendStr("    on ce_func.request_id = r.id \r\n");
bSqlStr.AppendStr("    and ce_func.name = 'f_candidate_function' \r\n");

bSqlStr.AppendStr("where r.person_id = " + Int(iPersonID) + " \r\n");

aTempArray = XQuery("sql:" + bSqlStr.GetStr());
