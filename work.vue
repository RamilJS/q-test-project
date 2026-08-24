alert("start");
try
{
    var r = XQuery("sql:select 1 as x");
    alert("OK: typeof=" + typeof r + " value=" + r);
}
catch (e)
{
    alert("FAIL: " + e);
    var k, keys = "";
    for (k in e) { keys += k + "=" + e[k] + "; "; }
    alert("keys ошибки: " + keys);
}
