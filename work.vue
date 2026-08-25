var oDoc = tools.open_doc(collaboratorId);
var oTE = oDoc.TopElem;
try {
    oTE.custom_elems.ObtainChildByKey("f_position_common_id").value = testValue;
    oDoc.BindToDb(DefaultDb);
    oDoc.Save();
    alert("OK: записано и сохранено");
} catch (e) {
    alert("FAIL: " + e);
}
