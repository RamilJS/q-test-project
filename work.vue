function GetActiveCollaboratorPositions()
{
    LogAlert(1, "GetActiveCollaboratorPositions(). НАЧАЛО");
    var sqlText, rows;

    sqlText = "";
    sqlText = sqlText + "select id, position_name\r\n";
    sqlText = sqlText + "from collaborators\r\n";
    sqlText = sqlText + "where is_dismiss = 0\r\n";
    sqlText = sqlText + "  and position_name is not null\r\n";
    sqlText = sqlText + "  and ltrim(rtrim(position_name)) != ''\r\n";
    alert("sqlText = " + sqlText);

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
