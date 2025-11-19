// параметры (подставь свои / бери из Param)
var personId = OptInt(Param.personId || curUserID, 0);
var requestTypeId = (Param.requestTypeId || ""); // например "0x63D8B73554856B02"

// строим SQL
var bSqlStr = new Binary();

bSqlStr.AppendStr("SELECT \r\n");
bSqlStr.AppendStr("  r.id, \r\n");
bSqlStr.AppendStr("  r.create_date, \r\n");
bSqlStr.AppendStr("  r.request_type_id, \r\n");
// правильно: двойные одинарные кавычки вокруг имени custom_elem
bSqlStr.AppendStr("  r.[data].value('(request/custom_elems/custom_elem[name=''f_vacancy_name'']/value)[1]', 'varchar(max)') AS f_vacancy_name, \r\n");
bSqlStr.AppendStr("  r.[data].value('(request/custom_elems/custom_elem[name=''f_candidate_function'']/value)[1]', 'varchar(max)') AS f_candidate_function \r\n");
bSqlStr.AppendStr("FROM requests r \r\n");
bSqlStr.AppendStr("WHERE r.person_id = " + Int(personId) + " \r\n");

if (requestTypeId != "") {
  bSqlStr.AppendStr("  AND r.request_type_id = " + SqlLiteral(requestTypeId) + " \r\n");
}

bSqlStr.AppendStr("ORDER BY r.create_date DESC \r\n");

// Выполнение
var aResults = XQuery("sql:" + bSqlStr.GetStr());
