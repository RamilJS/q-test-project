// personId и requestTypeId — входные параметры (если requestTypeId == '' — не фильтруем по типу)
var personId = OptInt(Param.personId || curUserID, 0);
var requestTypeId = (Param.requestTypeId || ""); // пример: 7194701844403546882

var bSqlStr = new Binary();

bSqlStr.AppendStr("SELECT \r\n");
bSqlStr.AppendStr("  r.id, \r\n");
bSqlStr.AppendStr("  r.create_date, \r\n");
bSqlStr.AppendStr("  r.request_type_id, \r\n");
// вытаскиваем кастомное поле f_vacancy_name из XML-хранилища колонки data
bSqlStr.AppendStr("  r.[data].value('(request/custom_elems/custom_elem[name=\"f_vacancy_name\"]/value)[1]', 'varchar(max)') AS f_vacancy_name, \r\n");
// вытаскиваем кастомное поле f_candidate_function
bSqlStr.AppendStr("  r.[data].value('(request/custom_elems/custom_elem[name=\"f_candidate_function\"]/value)[1]', 'varchar(max)') AS f_candidate_function \r\n");
bSqlStr.AppendStr("FROM requests r \r\n");
bSqlStr.AppendStr("WHERE r.person_id = " + Int(personId) + " \r\n");

if (requestTypeId != "") {
  // если requestTypeId передано — фильтруем по нему
  bSqlStr.AppendStr("  AND r.request_type_id = " + SqlLiteral(requestTypeId) + " \r\n");
}

bSqlStr.AppendStr("ORDER BY r.create_date DESC \r\n");

// выполняем
var aResults = XQuery("sql:" + bSqlStr.GetStr());
