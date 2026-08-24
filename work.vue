alert("test3 start");
try
{
    var sql_str = "SELECT cs.id\r\n";
    sql_str += "FROM collaborators cs\r\n";
    sql_str += "WHERE cs.is_dismiss = 0\r\n";

    var tArr = XQuery("sql:" + sql_str);

    var cnt = 0;
    for (elem in tArr)
    {
        cnt = cnt + 1;
        if (cnt <= 3) { alert("elem.id = " + elem.id); }
    }
    alert("OK test3: cnt=" + cnt);
}
catch (e3)
{
    alert("FAIL test3: " + e3);
}
