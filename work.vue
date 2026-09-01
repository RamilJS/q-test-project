alert("--- XQuery('cc_collaborator_mircodes') ---");
try
{
    var rows = XQuery("cc_collaborator_mircodes");
    var count = ArrayCount(ArraySelectAll(rows));
    alert("OK: XQuery('cc_collaborator_mircodes') -- count=" + count);
}
catch (eq)
{
    alert("FAIL: XQuery('cc_collaborator_mircodes') -- " + eq);
}
