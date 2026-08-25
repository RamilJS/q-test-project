var oDoc = tools.open_doc(collaboratorId);
var oTE = oDoc.TopElem;
try {
    oTE.custom_elems.ObtainChildByKey("f_position_common_id").value = testValue;
    oDoc.Save(); // без BindToDb — документ уже привязан к БД
    alert("OK: записано и сохранено");
} catch (e) {
    alert("FAIL: " + e);
}
