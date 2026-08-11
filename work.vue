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
