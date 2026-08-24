function GetActiveCollaboratorPositions()
{
    LogAlert(1, "GetActiveCollaboratorPositions(). НАЧАЛО");
    var bSqlStr, rows, sqlText;

    try { bSqlStr = new Binary(); alert("OK: new Binary()"); }
    catch (e1) { alert("FAIL: new Binary() -- " + e1); return []; }

    try
    {
        bSqlStr.AppendStr("select id, position_name\r\n");
        bSqlStr.AppendStr("from collaborators\r\n");
        bSqlStr.AppendStr("where is_dismiss = 0\r\n");
        bSqlStr.AppendStr("  and position_name is not null\r\n");
        bSqlStr.AppendStr("  and ltrim(rtrim(position_name)) != ''\r\n");
        alert("OK: AppendStr x5");
    }
    catch (e2) { alert("FAIL: AppendStr -- " + e2); return []; }

    try { sqlText = bSqlStr.GetStr(); alert("OK: GetStr() = " + sqlText); }
    catch (e3) { alert("FAIL: GetStr() -- " + e3); return []; }

    try { rows = XQuery("sql:" + sqlText); alert("OK: XQuery() typeof=" + typeof rows); }
    catch (e4) { alert("FAIL: XQuery() -- " + e4); return []; }

    try { alert("rows.length = " + rows.length); }
    catch (e5) { alert("FAIL: rows.length -- " + e5); }

    try { alert("rows[0] = " + rows[0]); }
    catch (e6) { alert("FAIL: rows[0] -- " + e6); }

    try { alert("rows.Count = " + rows.Count); }
    catch (e7) { alert("FAIL: rows.Count -- " + e7); }

    try { alert("rows.item -- " + typeof rows.item); }
    catch (e8) { alert("FAIL: rows.item -- " + e8); }

    try
    {
        var key, keysFound = "";
        for (key in rows) { keysFound += key + ", "; }
        alert("for..in rows keys: " + keysFound);
    }
    catch (e9) { alert("FAIL: for..in rows -- " + e9); }

    return rows;
}
