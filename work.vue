iPersonID = curUserID;  // текущий пользователь

bSqlStr = new Binary();

bSqlStr.AppendStr("select \r\n");
bSqlStr.AppendStr("    r.id, \r\n");
bSqlStr.AppendStr("    r.person_id, \r\n");
bSqlStr.AppendStr("    r.create_date, \r\n");
bSqlStr.AppendStr("    ce1.value as f_vacancy_name, \r\n");
bSqlStr.AppendStr("    ce2.value as f_candidate_function \r\n");
bSqlStr.AppendStr("from requests r \r\n");

bSqlStr.AppendStr("left join request_custom_elems ce1 \r\n");
bSqlStr.AppendStr("    on ce1.request_id = r.id and ce1.name = 'f_vacancy_name' \r\n");

bSqlStr.AppendStr("left join request_custom_elems ce2 \r\n");
bSqlStr.AppendStr("    on ce2.request_id = r.id and ce2.name = 'f_candidate_function' \r\n");

bSqlStr.AppendStr("where r.person_id = " + Int(iPersonID) + "\r\n");

aTempArray = XQuery("sql:" + bSqlStr.GetStr());

