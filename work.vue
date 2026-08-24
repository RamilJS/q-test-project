alert("test4 start");
try
{
    var sql_str = "SELECT cs.id, cs.position_name\r\n";
    sql_str += "FROM collaborators cs\r\n";
    sql_str += "WHERE cs.is_dismiss = 0\r\n";
    sql_str += "  AND cs.position_name IS NOT NULL\r\n";
    sql_str += "  AND LTRIM(RTRIM(cs.position_name)) != ''\r\n";

    var tArr = XQuery("sql:" + sql_str);

    var cnt = 0;
    for (elem in tArr)
    {
        cnt = cnt + 1;
        if (cnt <= 3) { alert("elem.id=" + elem.id + " pos=" + elem.position_name); }
    }
    alert("OK test4: cnt=" + cnt);
    alert("test4 tArr.length = " + tArr.length);
}
catch (e4)
{
    alert("FAIL test4: " + e4);
}
