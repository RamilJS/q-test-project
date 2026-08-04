bIsSamePerson = OptInt(objDocSec.custom_elems.ObtainChildByKey("f_manager_id").value, 0)
    == OptInt(objDocSec.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value, 0)
    && OptInt(objDocSec.custom_elems.ObtainChildByKey("f_manager_id").value, 0) != 0;

sAcceptActionId = bIsSamePerson ? "02_03" : "01_02";
sDeclineActionId = bIsSamePerson ? "02_06" : "01_05";

sAcceptBody = "Согласовать заявку\n\nНе удаляйте и не изменяйте информацию ниже\n----service_data----\n" + objDocSecID + "|" + sAcceptActionId + "|" + objDocID + "\n----service_data----";
sDeclineBody = "Отклонить заявку\n\nНе удаляйте и не изменяйте информацию ниже\n----service_data----\n" + objDocSecID + "|" + sDeclineActionId + "|" + objDocID + "\n----service_data----";
