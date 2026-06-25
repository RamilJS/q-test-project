oReqDoc = tools.new_doc_by_name("request");
oReqDocTE = oReqDoc.TopElem;

//oReqDocTE.object_id = oOptions.id;
oReqDocTE.object_id = 0x68790E4B5FD73CE8;

//oReqDocTE.person_id = Int(sID);
oReqDocTE.person_id = 0x593015312B3B5F71;

alert("RAMIl-33");

//tools.common_filling( 'request_type', oReqDocTE, 7528328911430055448 );
tools.common_filling('request_type', oReqDocTE, 0x6879FF4216177618);

//tools.common_filling( 'collaborator', oReqDocTE, oReqDocTE.person_id );
tools.common_filling('collaborator', oReqDocTE, 0x593015312B3B5F71);

//tools.object_filling( 'education_method', oReqDocTE, oReqDocTE.object_id);
tools.object_filling('education_method', oReqDocTE, 0x68790E4B5FD73CE8);

alert("RAMIl-34");

//oReqDocTE.custom_elems.ObtainChildByKey("f_vtbl_manager_mail").value = oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_manager_mail").value;
oReqDocTE.custom_elems.ObtainChildByKey("f_vtbl_manager_mail").value =
'Уважаемый(ая) ##фио_руководителя##!<br /><br />При бюджетировании внешнего обучения на <strong>##текущий_год##</strong> год и/или при постановке целей по развитию на <strong>##текущий_год##</strong> год для Вашего сотрудника ##фио_сотрудника## было запланировано участие в обучающем мероприятии <strong>"##название_мероприятия##"</strong>.<br /><br />Данный курс запланирован к проведению в следующие даты:<br />##список_дат##<br />Ориентировочная стоимость – <strong>##ориентировочная_стоимость##</strong> Итоговая стоимость будет зависеть от количества участников в группе.<br /><br />Этапы согласования:<br />##список_согласования##<br />Ссылка на заявку: ##ссылка_на_заявку##<br /><br /><strong>Если участие сотрудника в данном обучении неактуально</strong>, то нажмите кнопку<br />##кнопка_отклонения##.<br /><br /><strong>Если участие сотрудника в данном обучении актуально</strong>, то нажмите кнопку<br />##кнопка_подтверждения##, либо проигнорируйте письмо (в этом случае участие будет считаться согласованным Вами и через ##количество_дней_руководителя## рабочих дня Учебный портал переведет заявку на следующий этап).<br /><br /><br />В случае вопросов, пожалуйста, обращайтесь к куратору курса: ##отвественный_сотрудник##<br />Не отвечайте на это письмо. Для согласования задачи используйте ссылки в письме.';

alert("RAMIl-35");

//oReqDocTE.custom_elems.ObtainChildByKey("f_vtbl_mir_code_manager_mail").value = oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_mir_code_manager_mail").value;
oReqDocTE.custom_elems.ObtainChildByKey("f_vtbl_mir_code_manager_mail").value =
'Уважаемый(ая) ##фио_руководителя_мир_кода##!<br /><br />При бюджетировании внешнего обучения на <strong>##текущий_год##</strong> год и/или при постановке целей по развитию на <strong>##текущий_год##</strong> год для Вашего сотрудника ##фио_сотрудника## было запланировано участие в обучающем мероприятии <strong>"##название_мероприятия##"</strong>.<br /><br />Данный курс запланирован к проведению в следующие даты:<br />##список_дат##<br />Ориентировочная стоимость – <strong>##ориентировочная_стоимость##</strong> Итоговая стоимость будет зависеть от количества участников в группе.<br /><br />Этапы согласования:<br />##список_согласования##<br />Ссылка на заявку: ##ссылка_на_заявку##<br /><br /><strong>Если участие сотрудника в данном обучении неактуально</strong>, то нажмите кнопку<br />##кнопка_отклонения##.<br /><br /><strong>Если участие сотрудника в данном обучении актуально</strong>, то нажмите кнопку<br />##кнопка_подтверждения##, либо проигнорируйте письмо (в этом случае участие будет считаться согласованным Вами и через ##количество_дней_руководителя_мир_кода## рабочих дня Учебный портал переведет заявку на следующий этап).<br /><br /><br />В случае вопросов, пожалуйста, обращайтесь к куратору курса: ##отвественный_сотрудник##<br />Не отвечайте на это письмо. Для согласования задачи используйте ссылки в письме.';

alert("RAMIl-35");

//oReqDocTE.custom_elems.ObtainChildByKey("f_vtbl_collaborator_mail").value = oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_collaborator_mail").value;
oReqDocTE.custom_elems.ObtainChildByKey("f_vtbl_collaborator_mail").value =
'Добрый день, ##фио_сотрудника##!<br /><br />Ваше участие в мероприятии "##название_мероприятия##" планировалось в процессе бюджетирования внешнего обучения на ##текущий_год## год и/или при постановке целей по развитию на ##текущий_год## год в рамках УЭД.<br /><br />Данное письмо подтверждает, что Ваш руководитель/владелец MIR кода подтвердил/(и) возможность Вашего участия в данном мероприятии.<br /><br />Данный курс запланирован к проведению в следующие даты:<br />##список_дат##<br />Вам необходимо выбрать подходящие даты обучения и пройти регистрацию в течение <strong>3-х рабочих дней с момента получения данного письма</strong>.<br /><br />После регистрации (можно выбрать только 1 группу) мероприятие будет автоматически добавлено в Ваш календарь.<br /><br />Для выбора дат и регистрации перейдите по ссылке: ##ссылка_на_заявку##<br /><br />Если Вы не планируете принимать участие в мероприятии, пожалуйста, нажмите на кнопку:<br />##кнопка_отклонения##.<br /><br />В случае вопросов, пожалуйста, обращайтесь к куратору курса: ##отвественный_сотрудник##<br /><br /><br />Не отвечайте на это письмо. Для согласования задачи используйте ссылки в письме.';

alert("RAMIl-36");

//oReqDocTE.custom_elems.ObtainChildByKey("f_vtbl_auto_accept_manager_days_count").value = oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_auto_accept_manager_days_count").value;
oReqDocTE.custom_elems.ObtainChildByKey("f_vtbl_auto_accept_manager_days_count").value = "0";

alert("RAMIl-37");

//oReqDocTE.custom_elems.ObtainChildByKey("f_vtbl_auto_accept_mir_code_manager_days_count").value = oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_auto_accept_mir_code_manager_days_count").value;
oReqDocTE.custom_elems.ObtainChildByKey("f_vtbl_auto_accept_mir_code_manager_days_count").value = "0";

alert("RAMIl-38");

//oReqDocTE.custom_elems.ObtainChildByKey("f_vtbl_desc").value = oDoc.TopElem.desc;
oReqDocTE.custom_elems.ObtainChildByKey("f_vtbl_desc").value =
"<p>Выбор функциональной области Сильные и слабые стороны Возможности и угрозы среды Выработка решений</p>";

alert("RAMIl-39");

oReqDoc.BindToDb(DefaultDb);

alert("RAMIl-40");
alert(oReqDoc.TopElem.Xml);

oReqDoc.Save();

alert("RAMIl-41");
