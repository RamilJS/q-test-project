iPersonID = curUserID;  // текущий пользователь
requestTypeID = "7194701844403546882";

sSearchStr = Trim(search);

bSqlStr = new Binary();

bSqlStr.AppendStr("with fcte as (\r\n");
bSqlStr.AppendStr("    select r.id,\r\n");
bSqlStr.AppendStr("           r.create_date,\r\n");
bSqlStr.AppendStr("           r.request_type_id,\r\n");
bSqlStr.AppendStr("           r.data.value('(*/custom_elems/custom_elem[name=''f_vacancy_name'']/value)[1]', 'varchar(max)') as f_vacancy_name,\r\n");
bSqlStr.AppendStr("           r.data.value('(*/custom_elems/custom_elem[name=''f_candidate_function'']/value)[1]', 'varchar(max)') as f_candidate_function\r\n");
bSqlStr.AppendStr("    from requests r\r\n");
bSqlStr.AppendStr("    where r.person_id = " + OptInt(iPersonID) + "\r\n");
bSqlStr.AppendStr("      and r.request_type_id = " + OptInt(requestTypeID) + "\r\n");
bSqlStr.AppendStr(")\r\n");

bSqlStr.AppendStr("select id, create_date, request_type_id\r\n");
bSqlStr.AppendStr("from fcte\r\n");

if (sSearchStr != '') {
    sSearchStr = SqlLiteral('%' + sSearchStr + '%'); // защита от SQL-инъекций
    bSqlStr.AppendStr("where f_vacancy_name like " + sSearchStr + "\r\n");
    bSqlStr.AppendStr("   or f_candidate_function like " + sSearchStr + "\r\n");
}

aTempArray = XQuery("sql:" + bSqlStr.GetStr());
