iPersonID = curUserID;
requestTypeID = "7194701844403546882";

sSearchStr = Trim(search);

bSqlStr = new Binary();

bSqlStr.AppendStr("with fcte as (\r\n");
bSqlStr.AppendStr("    select \r\n");
bSqlStr.AppendStr("        rs.id,\r\n");
bSqlStr.AppendStr("        rs.create_date,\r\n");
bSqlStr.AppendStr("        rs.request_type_id,\r\n");
bSqlStr.AppendStr("        rs.data.value('(request/custom_elems/custom_elem[name=''f_vacancy_name'']/value)[1]', 'varchar(max)') as f_vacancy_name,\r\n");
bSqlStr.AppendStr("        rs.data.value('(request/custom_elems/custom_elem[name=''f_candidate_quality'']/value)[1]', 'varchar(max)') as f_candidate_quality\r\n");
bSqlStr.AppendStr("    from requests rs\r\n");
bSqlStr.AppendStr("    where rs.person_id = " + OptInt(iPersonID) + "\r\n");
bSqlStr.AppendStr("      and rs.request_type_id = " + OptInt(requestTypeID) + "\r\n");
bSqlStr.AppendStr(")\r\n");

bSqlStr.AppendStr("select *\r\n");
bSqlStr.AppendStr("from fcte\r\n");

if (sSearchStr != '') {
    sSearchStr = '%' + sSearchStr + '%';
    bSqlStr.AppendStr("where fcte.f_vacancy_name like " + SqlLiteral(sSearchStr) + "\r\n");
    bSqlStr.AppendStr("   or fcte.f_candidate_quality like " + SqlLiteral(sSearchStr) + "\r\n");
}
