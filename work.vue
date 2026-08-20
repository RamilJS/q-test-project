accept_budget_mircode_manager_accept_2

<%
function getMirCodeObject(sMirCode){
    var aMircodes = ArrayDirect(ArraySelect(String(sMirCode).split("|"), "This != ''"));
    for (i = 0; i < ArrayCount(aMircodes); i++){
        aMircodes[i] = ArrayDirect(ArraySelect(String(aMircodes[i]).split("#"), "This != ''"));
        aMircodes[i][1] = OptInt(aMircodes[i][1], 0);
    }
    oResult = new Object();
    oResult.mir_codes = [];
    oResult.main_mircode = new Object();
    oResult.main_mircode.code = "-";
    oResult.main_mircode.percent = 0;
    oResult.main_mircode.boss = "";
    for (i = 0; i < ArrayCount(aMircodes); i++){
        sBossMircode = "";
        iBossMircodeID = 0;
        oFindMircode = ArrayOptFirstElem(XQuery("for $elem in cc_mir_codes where $elem/name=" + XQueryLiteral(aMircodes[i][0]) + " return $elem"));
        if (oFindMircode != undefined && oFindMircode.owner_id > 0) {
            try {
                sBossMircode = oFindMircode.owner_id.ForeignElem.fullname;
                iBossMircodeID = oFindMircode.owner_id;
            }
            catch(_e){}
        }
        oResult.mir_codes.push({
            "code": aMircodes[i][0],
            "percent": aMircodes[i][1],
            "boss": sBossMircode,
            "boss_id": iBossMircodeID
        })
    }
    if (ArrayCount(oResult.mir_codes) > 0) {
        oResult.mir_codes = ArraySort(oResult.mir_codes, "percent", "-");
        oResult.main_mircode = oResult.mir_codes[0];
    }
    return oResult;
}
%>
<style>
table, table tr, table th, table td{
border-collapse: collapse;
border: 1px solid #000;
padding: 5px;
}
</style>
<%
sAcceptBody = "Согласовать заявку\n\nНе удаляйте и не изменяйте информацию ниже\n----service_data----\n" + objDocSecID + "|02_03|" + objDocID + "\n----service_data----";
sDeclineBody = "Отклонить заявку\n\nНе удаляйте и не изменяйте информацию ниже\n----service_data----\n" + objDocSecID + "|02_06|" + objDocID + "\n----service_data----";
aReplaceDictionary = [];
aReplaceDictionary.push(["##фио_сотрудника##", objDocSec.person_id.ForeignElem.fullname]);
try {
    aReplaceDictionary.push(["##фио_руководителя##", tools.open_doc(OptInt(objDocSec.custom_elems.ObtainChildByKey("f_manager_id").value, 0)).TopElem.fullname]);
}
catch(_e){
    aReplaceDictionary.push(["##фио_руководителя##", _e]);
}
try {
    aReplaceDictionary.push(["##фио_руководителя_мир_кода##", tools.open_doc(OptInt(objDocSec.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value, 0)).TopElem.fullname]);
}
catch(_e){
    aReplaceDictionary.push(["##фио_руководителя_мир_кода##", _e]);
}
try {
    oTempUser = tools.open_doc(OptInt(objDocSec.custom_elems.ObtainChildByKey("f_main_collaborator").value, 0)).TopElem
    sTempString = oTempUser.fullname + " (" + oTempUser.email + ")";
    aReplaceDictionary.push(["##отвественный_сотрудник##", sTempString]);
}
catch(_e){
    aReplaceDictionary.push(["##отвественный_сотрудник##", "ed.support@vtb-leasing.ru"]);
}
aReplaceDictionary.push(["##текущий_год##", Year(Date())]);
aReplaceDictionary.push(["##название_мероприятия##", objDocSec.object_id.ForeignElem.name]);
aReplaceDictionary.push(["##ориентировочная_стоимость##", objDocSec.custom_elems.ObtainChildByKey("f_ever_cost").value]);
oEducationMethodTE = tools.open_doc(objDocSec.object_id).TopElem;
aReplaceDictionary.push(["##количество_дней_руководителя##", oEducationMethodTE.custom_elems.ObtainChildByKey("f_vtbl_auto_accept_manager_days_count").value]);
aReplaceDictionary.push(["##количество_дней_руководителя_мир_кода##", oEducationMethodTE.custom_elems.ObtainChildByKey("f_vtbl_auto_accept_mir_code_manager_days_count").value]);
aReplaceDictionary.push(["##ссылка_на_заявку##", ("<a href=\"https://als-wt/_wt/" + objDocSecID + "\">https://als-wt/_wt/" + objDocSecID + "</a>")]);
aReplaceDictionary.push(["##кнопка_подтверждения##", ("<a href=\"mailto:sdo@vtb-leasing.ru?subject=" + UrlEncode("Согласовать заявку на потребность в обучении") + "&body=" + UrlEncode(sAcceptBody) + "\">\"Подтвердить участие\"</a>")]);
aReplaceDictionary.push(["##кнопка_отклонения##", ("<a href=\"mailto:sdo@vtb-leasing.ru?subject=" + UrlEncode("Отклонить заявку на потребность в обучении") + "&body=" + UrlEncode(sDeclineBody) + "\">\"Отклонить участие\"</a>")]);

// ЗАМЕНЕНО: вместо списка конкретных дат — формулировка "даты уточняются"
aReplaceDictionary.push(["##список_дат##", "Даты курса в настоящий момент на согласовании и будут сообщены Вам дополнительно."]);

try {
    oManagerTE = tools.open_doc(OptInt(objDocSec.custom_elems.ObtainChildByKey("f_manager_id").value, 0)).TopElem;
    oMirCodeManagerTE = tools.open_doc(OptInt(objDocSec.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value, 0)).TopElem;
    sTempString = "<table><tr><th>Роль</th><th>ФИО</th><th>Должность</th><th>Виза</th></tr>";
    sTempString += "<tr><td>Руководитель</td><td>" + oManagerTE.fullname + "</td><td>" + oManagerTE.position_name + "</td><td>" + (objDocSec.custom_elems.ObtainChildByKey("f_manager_id").value != objDocSec.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value ? "Согласовано" : "") + "</td></tr>";
    sTempString += "<tr><td>Владелец Мир-кода</td><td>" + oMirCodeManagerTE.fullname + "</td><td>" + oMirCodeManagerTE.position_name + "</td><td></td></tr>";
    sTempString += "</table>"
    aReplaceDictionary.push(["##список_согласования##", sTempString]);
}
catch(_e){
    aReplaceDictionary.push(["##список_согласования##", _e]);
}
sTemplateString = objDocSec.custom_elems.ObtainChildByKey("f_vtbl_mir_code_manager_mail").value;
for (aReplaceElem in aReplaceDictionary) {
    sTemplateString = StrReplace(sTemplateString, aReplaceElem[0], aReplaceElem[1])
}
sTemplateString = "<div>" + sTemplateString + "</div>";
sReplaceStr = ", либо проигнорируйте письмо (в этом случае участие будет считаться согласованным Вами и через 0 рабочих дня Учебный портал переведет заявку на следующий этап)";
sTemplateString = StrReplace(sTemplateString, sReplaceStr, "");
Response.Write(sTemplateString);

aAnotherPeoples = [];
oEdDoc = tools.open_doc(objDocSec.object_id).TopElem;
peoplesArrs = XQuery("for $elem in collaborators where MatchSome($elem/id, (" + StrReplace(oEdDoc.custom_elems.ObtainChildByKey("f_vtbl_collaborators").value, ";", ",") + ")) and $elem/id != " + objDocSec.person_id + " return $elem");
for (oUser in peoplesArrs) {
    try {
        userDocTE = tools.open_doc(oUser.id).TopElem;
        oMirCode = getMirCodeObject(String(userDocTE.custom_elems.ObtainChildByKey("f_mir_codes").value));
        if (Int(oMirCode.main_mircode.boss_id) == OptInt(objDocSec.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value, 0)) {
            aAnotherPeoples.push(oUser);
        }
    }
    catch(_ex){
        curObject.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value = "";
    }
}

if (ArrayCount(aAnotherPeoples) > 0) {
    sTempString = "<p>Для информации – другие сотрудники Вашего Mir-code для подтверждения потребности в обучении по данной теме обучения: </p>";
    sTempString += "<table><tr><th>ФИО</th><th>Должность</th><th>Статус</th></tr>";
    for (elem in aAnotherPeoples) {
        sState = "На согласовании непосредственного руководителя";
        oReq = ArrayOptFirstElem( XQuery("for $elem in requests where $elem/request_type_id=7528328911430055448 and $elem/object_id =" + objDocSec.object_id + " and $elem/person_id = " + elem.id + " return $elem"));
        if (oReq == undefined || oReq.workflow_state != 'archive') {
            sTempString += "<tr><td>" + elem.fullname + "</td><td>" + elem.position_name + "</td><td>" + (oReq == undefined ? sState : oReq.workflow_state_name) + "</td></tr>"
        }
    }
    sTempString += "</table>";
    Response.Write(sTempString);
}
%>
<p style="margin:0px; font: 10pt calibri; color: #7F7F7F;"><i>Письмо сформировано автоматически, пожалуйста не отвечайте на него.<br/>Предупреждение: Полученная информация предназначена только для физического или юридического лица, которому она адресована, может являться конфиденциальной и содержать охраняемую тайну. Ознакомление, распространение, разглашение либо использование ее в любой форме, а также совершение любого действия, вытекающего из знания этой информации, физическим и/или юридическим лицом, не являющимся надлежащим получателем, запрещено.</i></p>
