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
              
               elemDates = ArrayOptFind(requestDoc.TopElem.custom_elems, "This.name=='f_dates'");
               if (elemDates != undefined)
               {
                              valueDates = elemDates.value;
                              strDate = StrCharRangePos(valueDates, 0, 10);
                              if (strDate.length == 10)
                              {
                                            startDate = Date(strDate);
                                            if (Day(startDate) == 1 && Month(startDate) == 1)
                                                           result = true;
                              }
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
               return oElem.send_flag;
}

aResponsibleMailArray = [];

aRequests = XQuery("for $elem in requests where $elem/request_type_id = 7528328911430055448 and $elem/workflow_state != 'archive' return $elem");
for (oRequest in aRequests) {
               oRequestDoc = tools.open_doc(oRequest.id);
               switch(oRequest.workflow_state) {
                              case "01": {
                                            collaboratorDoc = tools.open_doc(oRequestDoc.TopElem.person_id);
                                            manager = ArrayOptFind(collaboratorDoc.TopElem.func_managers, "This.is_native")
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

                                            // Пересчитываем владельца МИР-кода ДО принятия решения об уведомлении/переходе -
                                            // это важно, чтобы проверка совпадения ниже использовала актуальное значение
                                            oMirCode = getMirCodeObject(String(collaboratorDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_codes").value));
                                            if ((OptInt(oMirCode.main_mircode.boss_id, 0) != 0) && (oMirCode.main_mircode.boss_id != OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value,0))) {
                                                      oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value = OptInt(oMirCode.main_mircode.boss_id, 0);
                                                           oRequestDoc.Save();
                                            }

                                            iManagerID = OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_manager_id").value, 0);
                                            iMirCodeManagerID = OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value, 0);

                                            if (iManagerID != 0 && iManagerID == iMirCodeManagerID) {
                                                           // Руководитель совпал с владельцем МИР-кода (при создании заявки это было не так,
                                                           // но данные с тех пор изменились). Заявка не должна больше висеть на этапе 01 -
                                                           // переводим её на 02 принудительно, точно как это делает действие 00_00 при создании,
                                                           // и НЕ отправляем письмо руководителю (accept_budget_manager_accept),
                                                           // так как согласовывать нужно только один раз, на этапе 02.
                                                           oLogChild = oRequestDoc.TopElem.workflow_log_entrys.AddChild();
                                                           oLogChild.create_date = Date();
                                                           oLogChild.begin_state = '01';
                                                           oLogChild.finish_state = '02';
                                                           oLogChild.submited = 0;

                                                           oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_auto_date").value = Date();
                                                           oRequestDoc.TopElem.workflow_state = "02";
                                                           oRequestDoc.TopElem.workflow_state_name = "На согласовании владельца МИР-кода";
                                                           oRequestDoc.Save();

                                                           tools.create_notification('accept_budget_mircode_manager_accept', iMirCodeManagerID, '', oRequestDoc.DocID, undefined, oRequestDoc.TopElem);
                                            }
                                            else if (iManagerID != 0){
                                                           tools.create_notification('accept_budget_manager_accept', iManagerID, '', oRequestDoc.DocID, undefined, oRequestDoc.TopElem);
                                            }
                                            else {
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
                                            collaboratorDoc = tools.open_doc(oRequestDoc.TopElem.person_id);
                                            oMirCode = getMirCodeObject(String(collaboratorDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_codes").value));
                                            manager = ArrayOptFind(collaboratorDoc.TopElem.func_managers, "This.is_native")
                                            managerID = manager != undefined ? Int(manager.person_id) : 0;
                                           
                                            if (OptInt(oMirCode.main_mircode.boss_id, 0) != OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value,0) && (managerID != OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_manager_id").value,0))) {
                                                          
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
                                                      oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value = "";
                                                           oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_auto_date").value = Date();
                                                           oRequestDoc.Save();
                                            }
                                            else {
                                                           if (oMirCode.main_mircode.boss_id != OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value,0)) {
                                                                     oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value = Int(oMirCode.main_mircode.boss_id);
                                                                          oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_auto_date").value = Date();
                                                                          oRequestDoc.Save();
                                                                         
                                                           }
                                            }                                          
                                            if (OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value, 0) != 0) {
                                                           tools.create_notification('accept_budget_mircode_manager_accept', OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value, 0), '', oRequestDoc.DocID, undefined, oRequestDoc.TopElem);
                                            }
                                            else {
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
                                            if (getSendFlag(oRequest.object_id)) {
                                                           if (IsEventWithoutDates(oRequestDoc))
                                                                          tools.create_notification('accept_budget_collaborator_accept_notification_2', oRequest.person_id, '', oRequest.id);
                                                           else
                                                                          tools.create_notification('accept_budget_collaborator_accept_notification', oRequest.person_id, '', oRequest.id);
                                            }
                                            break;
                              }
               }
}
aResponsibleMailArray = ArraySort(aResponsibleMailArray, "This.object_name", "+");
if (ArrayCount(aResponsibleMailArray) > 0) {
               respArray = ArraySelectDistinct(ArrayExtract(aResponsibleMailArray, "This.responsible_person_id"), "This");
               for (iRespID in respArray) {
                              aTempArray = ArraySelect(aResponsibleMailArray, "This.responsible_person_id == iRespID");
                              tools.create_notification('accept_budget_responsible_none_manager', iRespID, tools.object_to_text(aTempArray, 'json'))
               }
}
