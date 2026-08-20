EnableLog("ramil-stage-debug", true);
alert("=== stage_processing START ===");

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
    oResult.main_mircode.boss_id = "";
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

function IsEventWithoutDates(requestDoc)
{
    result = false;
    try {
        oEducationMethodTE = tools.open_doc(requestDoc.TopElem.object_id).TopElem;
        valueDates = oEducationMethodTE.custom_elems.ObtainChildByKey("f_vtbl_dates").value;
        aDates = ParseJson(valueDates != "" ? valueDates : "[]");
        firstDate = ArrayOptFirstElem(aDates);
        if (firstDate != undefined) {
            startDate = Date(firstDate.start_date);
            if (Day(startDate) == 1 && Month(startDate) == 1)
                result = true;
        }
        alert("IsEventWithoutDates(" + requestDoc.DocID + ") = " + result);
    }
    catch(_e) {
        alert("IsEventWithoutDates(" + requestDoc.DocID + ") ERROR: " + _e);
        result = false;
    }
    return result;
}

aEventArray = [];
function getSendFlag(iEducationMethodID){
    iEducationMethodID = Int(iEducationMethodID);
    var oElem = ArrayOptFind(aEventArray, "This.id == iEducationMethodID");
    if (oElem == undefined){
        oElem = new Object;
        oElem.id = iEducationMethodID;
        oElem.send_flag = (ArrayCount(XQuery("for $elem in events where $elem/education_method_id=" + XQueryLiteral(iEducationMethodID) + " and $elem/start_date > " + XQueryLiteral(DateNewTime(Date())) + " return $elem")) > 0);
        aEventArray.push(oElem);
    }
    alert("getSendFlag(" + iEducationMethodID + ") = " + oElem.send_flag);
    return oElem.send_flag;
}

aResponsibleMailArray = [];

aRequests = XQuery("for $elem in requests where $elem/request_type_id = 7528328911430055448 and $elem/workflow_state != 'archive' return $elem");
alert("Найдено заявок: " + ArrayCount(aRequests));

for (oRequest in aRequests) {
    alert("--- Обрабатываю заявку id=" + oRequest.id + " state=" + oRequest.workflow_state + " person=" + oRequest.person_id);
    try {
        oRequestDoc = tools.open_doc(oRequest.id);
    }
    catch(_e) {
        alert("ОШИБКА open_doc заявки " + oRequest.id + ": " + _e);
        continue;
    }

    try {
        switch(oRequest.workflow_state) {
            case "01": {
                alert("case 01: заявка " + oRequest.id);
                try {
                    collaboratorDoc = tools.open_doc(oRequestDoc.TopElem.person_id);
                }
                catch(_e) {
                    alert("ОШИБКА open_doc сотрудника " + oRequestDoc.TopElem.person_id + ": " + _e);
                    break;
                }
                manager = ArrayOptFind(collaboratorDoc.TopElem.func_managers, "This.is_native")
                alert("manager найден: " + (manager != undefined));
                if (manager == undefined) {
                    oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_manager_id").value = "";
                    oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_auto_date").value = Date();
                    oRequestDoc.Save();
                }
                else {
                    if (manager.person_id != OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_manager_id").value,0)) {
                        oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_manager_id").value = Int(manager.person_id);
                        oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_auto_date").value = Date();
                        oRequestDoc.Save();
                    }
                }

                oMirCode = getMirCodeObject(String(collaboratorDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_codes").value));
                alert("oMirCode.main_mircode.boss_id = " + oMirCode.main_mircode.boss_id);
                if ((OptInt(oMirCode.main_mircode.boss_id, 0) != 0) && (oMirCode.main_mircode.boss_id != OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value,0))) {
                    oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value = OptInt(oMirCode.main_mircode.boss_id, 0);
                    oRequestDoc.Save();
                }

                iManagerID = OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_manager_id").value, 0);
                iMirCodeManagerID = OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value, 0);
                alert("iManagerID=" + iManagerID + " iMirCodeManagerID=" + iMirCodeManagerID);

                if (iManagerID != 0 && iManagerID == iMirCodeManagerID) {
                    alert("Ветка: руководитель = владелец МИР-кода, переход на 02");
                    oLogChild = oRequestDoc.TopElem.workflow_log_entrys.AddChild();
                    oLogChild.create_date = Date();
                    oLogChild.begin_state = '01';
                    oLogChild.finish_state = '02';
                    oLogChild.submited = 0;

                    oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_auto_date").value = Date();
                    oRequestDoc.TopElem.workflow_state = "02";
                    oRequestDoc.TopElem.workflow_state_name = "На согласовании владельца МИР-кода";
                    oRequestDoc.Save();

                    if (IsEventWithoutDates(oRequestDoc)) {
                        alert("Отправляю accept_budget_mircode_manager_accept_2 получателю " + iMirCodeManagerID);
                        tools.create_notification('accept_budget_mircode_manager_accept_2', iMirCodeManagerID, '', oRequestDoc.DocID, undefined, oRequestDoc.TopElem);
                    }
                    else {
                        alert("Отправляю accept_budget_mircode_manager_accept получателю " + iMirCodeManagerID);
                        tools.create_notification('accept_budget_mircode_manager_accept', iMirCodeManagerID, '', oRequestDoc.DocID, undefined, oRequestDoc.TopElem);
                    }
                }
                else if (iManagerID != 0){
                    alert("Ветка: письмо руководителю id=" + iManagerID);
                    if (IsEventWithoutDates(oRequestDoc)) {
                        alert("Отправляю accept_budget_manager_accept_2 получателю " + iManagerID);
                        tools.create_notification('accept_budget_manager_accept_2', iManagerID, '', oRequestDoc.DocID, undefined, oRequestDoc.TopElem);
                    }
                    else {
                        alert("Отправляю accept_budget_manager_accept получателю " + iManagerID);
                        tools.create_notification('accept_budget_manager_accept', iManagerID, '', oRequestDoc.DocID, undefined, oRequestDoc.TopElem);
                    }
                }
                else {
                    alert("Ветка: руководителя нет, заявка " + oRequest.id + " в проблемные");
                    aResponsibleMailArray.push({
                        responsible_person_id: Int(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_main_collaborator").value),
                        type: 'not_manager',
                        request_id: Int(oRequestDoc.DocID),
                        person_id: Int(oRequestDoc.TopElem.person_id),
                        person_fullname: String(oRequestDoc.TopElem.person_id.ForeignElem.fullname),
                        person_position: String(oRequestDoc.TopElem.person_id.ForeignElem.position_name),
                        workflow_state_name: String(oRequestDoc.TopElem.workflow_state_name),
                        object_id: Int(oRequestDoc.TopElem.object_id),
                        object_name: String(oRequestDoc.TopElem.object_id.ForeignElem.name)
                    });
                }

                break;
            }
            case "02": {
                alert("case 02: заявка " + oRequest.id);
                collaboratorDoc = tools.open_doc(oRequestDoc.TopElem.person_id);
                oMirCode = getMirCodeObject(String(collaboratorDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_codes").value));
                manager = ArrayOptFind(collaboratorDoc.TopElem.func_managers, "This.is_native")
                managerID = manager != undefined ? Int(manager.person_id) : 0;
                alert("managerID=" + managerID + " oMirCode.boss_id=" + oMirCode.main_mircode.boss_id + " f_manager_id=" + oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_manager_id").value + " f_mir_code_manager_id=" + oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value);

                if (OptInt(oMirCode.main_mircode.boss_id, 0) != OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value,0) && (managerID != OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_manager_id").value,0))) {
                    alert("Ветка: возврат на 01");
                    oLogChild = oRequestDoc.TopElem.workflow_log_entrys.AddChild();
                    oLogChild.create_date = Date();
                    oLogChild.begin_state = String(oRequestDoc.TopElem.workflow_state);
                    oLogChild.finish_state = '01';
                    oLogChild.submited = 0;

                    oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_auto_date").value = Date();

                    oRequestDoc.TopElem.workflow_state = "01";
                    oRequestDoc.TopElem.workflow_state_name = "На согласовании непосредственного руководителя";
                    oRequestDoc.Save();
                }
                else if (OptInt(oMirCode.main_mircode.boss_id, 0) == 0) {
                    alert("Ветка: владельца МИР-кода нет, обнуляем f_mir_code_manager_id");
                    oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value = "";
                    oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_auto_date").value = Date();
                    oRequestDoc.Save();
                }
                else {
                    if (oMirCode.main_mircode.boss_id != OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value,0)) {
                        alert("Ветка: обновляем f_mir_code_manager_id на " + oMirCode.main_mircode.boss_id);
                        oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value = Int(oMirCode.main_mircode.boss_id);
                        oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_auto_date").value = Date();
                        oRequestDoc.Save();
                    }
                }
                iFinalMirCodeManagerID = OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value, 0);
                alert("Итоговый f_mir_code_manager_id = " + iFinalMirCodeManagerID);
                if (iFinalMirCodeManagerID != 0) {
                    if (IsEventWithoutDates(oRequestDoc)) {
                        alert("Отправляю accept_budget_mircode_manager_accept_2 получателю " + iFinalMirCodeManagerID);
                        tools.create_notification('accept_budget_mircode_manager_accept_2', iFinalMirCodeManagerID, '', oRequestDoc.DocID, undefined, oRequestDoc.TopElem);
                    }
                    else {
                        alert("Отправляю accept_budget_mircode_manager_accept получателю " + iFinalMirCodeManagerID);
                        tools.create_notification('accept_budget_mircode_manager_accept', iFinalMirCodeManagerID, '', oRequestDoc.DocID, undefined, oRequestDoc.TopElem);
                    }
                }
                else {
                    alert("Ветка: владельца МИР-кода нет, заявка " + oRequest.id + " в проблемные");
                    aResponsibleMailArray.push({
                        responsible_person_id: Int(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_main_collaborator").value),
                        type: 'not_mir_code_manager',
                        request_id: Int(oRequestDoc.DocID),
                        person_id: Int(oRequestDoc.TopElem.person_id),
                        person_fullname: String(oRequestDoc.TopElem.person_id.ForeignElem.fullname),
                        person_position: String(oRequestDoc.TopElem.person_id.ForeignElem.position_name),
                        workflow_state_name: String(oRequestDoc.TopElem.workflow_state_name),
                        object_id: Int(oRequestDoc.TopElem.object_id),
                        object_name: String(oRequestDoc.TopElem.object_id.ForeignElem.name)
                    });
                }
                break;
            }
            case "03": {
                alert("case 03: заявка " + oRequest.id + " object_id=" + oRequest.object_id);
                if (getSendFlag(oRequest.object_id)) {
                    if (IsEventWithoutDates(oRequestDoc)) {
                        alert("Отправляю accept_budget_collaborator_accept_notification_2 получателю " + oRequest.person_id);
                        tools.create_notification('accept_budget_collaborator_accept_notification_2', oRequest.person_id, '', oRequest.id);
                    }
                    else {
                        alert("Отправляю accept_budget_collaborator_accept_notification получателю " + oRequest.person_id);
                        tools.create_notification('accept_budget_collaborator_accept_notification', oRequest.person_id, '', oRequest.id);
                    }
                }
                else {
                    alert("getSendFlag=false, письмо НЕ отправляется для заявки " + oRequest.id);
                }
                break;
            }
            default: {
                alert("Неизвестный workflow_state для заявки " + oRequest.id + ": " + oRequest.workflow_state);
                break;
            }
        }
    }
    catch(_eOuter) {
        alert("ОБЩАЯ ОШИБКА обработки заявки " + oRequest.id + ": " + _eOuter);
    }
}

aResponsibleMailArray = ArraySort(aResponsibleMailArray, "This.object_name", "+");
alert("Проблемных заявок (без ответственного): " + ArrayCount(aResponsibleMailArray));
if (ArrayCount(aResponsibleMailArray) > 0) {
    respArray = ArraySelectDistinct(ArrayExtract(aResponsibleMailArray, "This.responsible_person_id"), "This");
    for (iRespID in respArray) {
        aTempArray = ArraySelect(aResponsibleMailArray, "This.responsible_person_id == iRespID");
        alert("Отправляю accept_budget_responsible_none_manager получателю " + iRespID);
        tools.create_notification('accept_budget_responsible_none_manager', iRespID, tools.object_to_text(aTempArray, 'json'))
    }
}

alert("=== stage_processing END ===");
