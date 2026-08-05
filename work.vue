было 

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

    // (А) — здесь сразу шлём письмо руководителю, БЕЗ учёта МИР-кода
    if (OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_manager_id").value, 0) != 0){
        tools.create_notification('accept_budget_manager_accept', OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_manager_id").value, 0), '', oRequestDoc.DocID, undefined, oRequestDoc.TopElem);
    }
    else {
        aResponsibleMailArray.push({ ... type: 'not_manager' ... });
    }

    // (Б) — пересчёт МИР-кода идёт ПОСЛЕ письма, и ни на что не влияет
    oMirCode = getMirCodeObject(String(collaboratorDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_codes").value));
    if ((OptInt(oMirCode.main_mircode.boss_id, 0) != 0) && (oMirCode.main_mircode.boss_id != OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value,0))) {
        oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value = OptInt(oMirCode.main_mircode.boss_id, 0);
        oRequestDoc.Save();
    }

    break;
}

стало

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

    // ▼▼▼ ПЕРЕНЕСЕНО СЮДА (было ниже, после письма) ▼▼▼
    oMirCode = getMirCodeObject(String(collaboratorDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_codes").value));
    if ((OptInt(oMirCode.main_mircode.boss_id, 0) != 0) && (oMirCode.main_mircode.boss_id != OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value,0))) {
        oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value = OptInt(oMirCode.main_mircode.boss_id, 0);
        oRequestDoc.Save();
    }

    // ▼▼▼ НОВОЕ: читаем актуальные значения в переменные ▼▼▼
    iManagerID = OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_manager_id").value, 0);
    iMirCodeManagerID = OptInt(oRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value, 0);

    // ▼▼▼ НОВОЕ: главная проверка и новая ветка if/else if/else ▼▼▼
    if (iManagerID != 0 && iManagerID == iMirCodeManagerID) {
        // руководитель = владелец МИР-кода -> принудительно переводим на 02
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
        // как раньше - письмо руководителю
        tools.create_notification('accept_budget_manager_accept', iManagerID, '', oRequestDoc.DocID, undefined, oRequestDoc.TopElem);
    }
    else {
        // как раньше - руководителя нет, в список проблемных
        aResponsibleMailArray.push({ ... type: 'not_manager' ... });
    }

    break;
}
