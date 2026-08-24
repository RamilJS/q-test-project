rows = XQuery("sql:" + bSqlStr.GetStr());
alert("typeof rows: " + typeof rows);
try { alert("rows.length = " + rows.length); } catch (e) { alert("rows.length упал: " + e); }
