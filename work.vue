function UpdateCollaboratorMirCodes(){
    //заполняет каталог "Сотрудник - МИР-код" (код cc_collaborator_mircode,
    //object_name: collaborator_mircode) теми же данными, что UpdateMirCodesForCollaborators()
    //пишет в f_mir_codes -- только не строкой в кастомное поле, а отдельными записями каталога.
    //Если пара collaborator_id+mir_code_id уже существует -- обновляется только percent.
    //Пары, которые перестали встречаться в текущей выгрузке 1С, не удаляются --
    //по той же логике, что мы не удаляем "пропавшие" коды из cc_mir_codes.
    var mirCodeNameToId, existingMirCodeElem, existingLinkKeyToId, existingLinkElem, sqlString,
        mirCodeElem, mirCodeArray, resultMirCodeArray, mirCode, mirCodeName, tMirCodeId, tLinkKey, tLinkDoc;

    //название кода -> id записи в cc_mir_codes (регистр приводим к верхнему, как и в UpdateMirCodes())
    mirCodeNameToId = new Object();
    for (existingMirCodeElem in XQuery('cc_mir_codes')) {
        mirCodeNameToId.SetProperty(StrUpperCase(Trim(String(existingMirCodeElem.name))), existingMirCodeElem.id);
    }

    //ключ "collaborator_id|mir_code_id" -> id уже существующей записи связи,
    //чтобы не плодить дубли и понимать, какую запись обновлять, а не создавать заново
    existingLinkKeyToId = new Object();
    for (existingLinkElem in XQuery('collaborator_mircodes')) {
        tLinkKey = String(existingLinkElem.collaborator_id) + "|" + String(existingLinkElem.mir_code_id);
        existingLinkKeyToId.SetProperty(tLinkKey, existingLinkElem.id);
    }

    //тот же самый запрос "сотрудник -> все коды его активных должностей", что и в UpdateMirCodesForCollaborators()
    sqlString = new Binary();
    sqlString.AppendStr("SELECT       [basic_collaborator_id] [id],\r\n");
    sqlString.AppendStr("                    STUFF((\r\n");
    sqlString.AppendStr("                                                 SELECT  ';'+ CAST([code] AS VARCHAR(MAX)) \r\n");
    sqlString.AppendStr("                                                 FROM    [positions]\r\n");
    sqlString.AppendStr("                                                 WHERE ([basic_collaborator_id] = [ps].[basic_collaborator_id])\r\n");
    sqlString.AppendStr("                                                 FOR XML PATH(''),TYPE\r\n");
    sqlString.AppendStr("                                   ).value('(./text())[1]','VARCHAR(MAX)')\r\n");
    sqlString.AppendStr("                                   ,1,1,''\r\n");
    sqlString.AppendStr("                    ) AS [pos_codes]\r\n");
    sqlString.AppendStr("FROM        [positions] [ps]\r\n");
    sqlString.AppendStr("WHERE      COALESCE([ps].[code], '') != ''\r\n");
    sqlString.AppendStr("                    AND       [ps].[is_position_finished] != 1\r\n");
    sqlString.AppendStr("GROUP BY              [basic_collaborator_id]");

    for (mirCodeElem in ArraySelectAll(XQuery("sql:" + sqlString.GetStr()))) {
        mirCodeArray = ArraySelect(globalObjects.array.mir_codes, ("This.ID == '" + ArrayMerge(String(mirCodeElem.pos_codes).split(";"), "This", "' || This.ID == '") + "'"));
        if (ArrayCount(mirCodeArray) > 0) {
            resultMirCodeArray = [];
            for (mirCode in mirCodeArray){
                resultMirCodeArray = ArrayUnion(resultMirCodeArray, mirCode.array)
            }
            for (mirCode in resultMirCodeArray) {
                mirCodeName = StrUpperCase(Trim(String(mirCode.MirCode)));
                if (!mirCodeNameToId.HasProperty(mirCodeName)) {
                    //в норме такого быть не должно -- UpdateMirCodes() уже должен создать
                    //заготовку по этому же globalObjects.array.mir_codes раньше в потоке.
                    //если всё же не нашли -- пропускаем пару, а не падаем с ошибкой
                    alert("UpdateCollaboratorMirCodes(). Не найден cc_mir_code для кода [" + mirCode.MirCode + "], collaborator id=" + mirCodeElem.id + ", пропуск");
                }
                else {
                    tMirCodeId = mirCodeNameToId[mirCodeName];
                    tLinkKey = String(Int(mirCodeElem.id)) + "|" + String(tMirCodeId);
                    if (existingLinkKeyToId.HasProperty(tLinkKey)) {
                        tLinkDoc = tools.open_doc(existingLinkKeyToId[tLinkKey]);
                    }
                    else {
                        tLinkDoc = tools.new_doc_by_name("collaborator_mircode");
                        tLinkDoc.BindToDb(DefaultDb);
                        tLinkDoc.TopElem.collaborator_id = Int(mirCodeElem.id);
                        tLinkDoc.TopElem.mir_code_id = tMirCodeId;
                    }
                    tLinkDoc.TopElem.percent = Real(mirCode.Percent);
                    tLinkDoc.Save();
                }
            }
        }
    }
}


if (globalObjects.update_flags.mir_codes) {
    alert('------------------------------------start update mir_codes------------------------------------');
    UpdateMirCodes();
    UpdateMirCodesForCollaborators();
    UpdateCollaboratorMirCodes();
    alert('------------------------------------end update mir_codes------------------------------------');
}
