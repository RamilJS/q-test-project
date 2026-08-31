function GetActivePositionsInUse()
{
    LogAlert(1, "GetActivePositionsInUse(). НАЧАЛО");
    var topClause, sqlText, rawRows, positionRows, row;
    topClause = "";
    if (TEST_TOP_N > 0)
    {
        topClause = "top " + TEST_TOP_N + " ";
    }
    sqlText = "";
    sqlText = sqlText + "select distinct " + topClause + "ps.id as position_id, ps.name as position_name\r\n";
    sqlText = sqlText + "from positions ps\r\n";
    sqlText = sqlText + "join collaborators cs on ps.basic_collaborator_id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss <> 1\r\n";
    sqlText = sqlText + "  and ps.is_position_finished <> 1\r\n";
    sqlText = sqlText + "  and ps.name is not null\r\n";
    sqlText = sqlText + "  and ltrim(rtrim(ps.name)) != ''\r\n";
    sqlText = sqlText + "  and ps.position_common_id is null\r\n";
    rawRows = XQuery("sql:" + sqlText);
    positionRows = [];
    for (row in rawRows)
    {
        positionRows.push(row);
    }
    LogAlert(1, "GetActivePositionsInUse(). Найдено должностей: " + positionRows.length);
    LogAlert(1, "GetActivePositionsInUse(). КОНЕЦ");
    return positionRows;
}
