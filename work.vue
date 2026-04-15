
EnableLog("Ramil", true)
function alert(elem) {
    LogEvent("Ramil", elem)
    return elem;
}
 
function getParam(sName, sDefault) {
    var sValue = PARAMETERS.GetOptProperty(sName);
    if (sDefault != undefined && (sValue == undefined || sValue == "")) {
        sValue = sDefault;
    }
    return sValue;
}
 
function getFormField(sName, sDefault) {
    var sValue = ArrayOptFind(aFormFields, ("This.name == " + XQueryLiteral(sName)));
    sValue = (sValue != undefined ? sValue.value : sValue);
    if (sDefault != undefined && (sValue == undefined || sValue == "")) {
        sValue = sDefault;
    }
    return sValue;
}
 
function getFormFieldDefault(sName, sDefault) {
    var sValue = ArrayOptFind(aFormFieldsDef, ("This.name == " + XQueryLiteral(sName)));
    sValue = (sValue != undefined ? sValue.value : sValue);
    if (sDefault != undefined && (sValue == undefined || sValue == "")) {
        sValue = sDefault;
    }
    return sValue;
}
 
// function createReportString(aCollaboratorIDArray, aAsessmentIDArray, dStartDate, dFinishDate){
//     //aCollaboratorIDArray - массив id сотрудников // один сотрудник
//     //aAsessmentIDArray - массив id курсов
//     //dStartDate - дата начала поиска попыток прохождений
//     //dFinishDate - дата окончания поиска попыток прохождений
//     reportStr = new Binary();
//     reportStr.AppendStr("<html xmlns:o=\"urn:schemas-microsoft-com:office:office\" xmlns:x=\"urn:schemas-microsoft-com:office:excel\" xmlns=\"http://www.w3.org/TR/REC-html40\"><head><meta http-equiv=\"Content-Type\" content=\"text/html; charset=utf-8\"/><meta name=\"ProgId\" content=\"Excel.Sheet\"/><meta name=\"Generator\" content=\"Microsoft Excel 11\"/><style>td.two-decimals {mso-number-format: \"0\\.00\"}</style></head><body><table border=\"1\" cellpadding=\"2\" cellspacing=\"0\">");
//     //сюда пишешт логику
 
//     reportStr.AppendStr("</table></body></html>");
//     return reportStr.GetStr();
// }
 
// --- Мой рабочий вариант ---
function createReportString(aCollaboratorIDArray, aAssessmentIDArray, dStartDate, dFinishDate)
{
    alert("старт функции createReportString");
 
   
    var aLearnings = [];
    var aQuestions = [];
 
    var raw_person_id;
    // for (i in aCollaboratorIDArray)
    for (var i = 0; i < ArrayCount(aCollaboratorIDArray); i++) {
        //var person_id = OptInt(aCollaboratorIDArray[i]);
 
        raw_person_id = aCollaboratorIDArray[i];
        //alert("raw_person_id = " + raw_person_id);
 
        if (raw_person_id == undefined)
            continue;
 
        raw_person_id = Trim(raw_person_id);
 
        if (raw_person_id == "")
            continue;
 
        var person_id = raw_person_id;
        alert("person_id = " + person_id);
       
 
        var raw_ass_id;
        var ass_id;
        var arr;
 
        //for (j in aAssessmentIDArray)
        for (var j = 0; j < ArrayCount(aAssessmentIDArray); j++)
        {
            //var ass_id = OptInt(aAssessmentIDArray[j]);
            alert("Массив IDs assesments " + aAssessmentIDArray);
 
            raw_ass_id = aAssessmentIDArray[j];
            //alert("ass_id = " + raw_ass_id);
 
            if (raw_ass_id == undefined)
                continue;
 
            raw_ass_id = Trim(raw_ass_id);
 
            if (raw_ass_id == "")
                continue;
 
            ass_id = raw_ass_id;
            alert("ass_id = " + ass_id);
 
           
           
            arr = XQuery(
                "for $l in test_learnings " +
                "where $l/person_id = " + XQueryLiteral(person_id) +
                " and $l/assessment_id = " + XQueryLiteral(ass_id) +
                " and $l/start_usage_date >= date('" + StrDate(dStartDate,false) + "')" +
                " and $l/start_usage_date <= date('" + StrDate(dFinishDate,false) + "')" +
                " return $l"
            );
 
            // var arr = XQuery(
            //     "for $l in (active_test_learnings, test_learnings) " +
            //     "where $l/person_id = " + person_id +
            //     " and $l/assessment_id = " + ass_id +
            //     " and $l/start_usage_date >= date('" + StrDate(dStartDate,false) + "')" +
            //     " and $l/start_usage_date <= date('" + StrDate(dFinishDate,false) + "')" +
            //     " return $l"
            // );
 
            alert("Array = " + arr);
            alert("ArrayCount(arr) = " + ArrayCount(arr));
 
            // var obj;
            // var l;
 
             for (l in arr)
            //for (var k = 0; k < ArrayCount(items); k++)
            {
                 //l = arr[k]; // тут падает
 
                 alert("LOOP l = " + l);
 
                obj = new Object();
 
                alert("l.id = " + l.id);
 
                obj.person_fullname = l.person_id.ForeignElem.fullname;
                obj.assessment_name = l.assessment_name;
 
                obj.person_code = l.person_id.ForeignElem.code;
               
                //obj.person_code = l.person_id.ForeignElem.person_position_code;
 
                obj.org = l.person_id.ForeignElem.org_name;
                obj.subdivision = l.person_id.ForeignElem.position_parent_name;
                obj.position = l.person_id.ForeignElem.position_name;
                obj.date = l.start_usage_date;
                obj.status = l.state_id.ForeignElem.name;
                obj.score = l.score;
                obj.max_score = l.max_score;
                // obj.assessment_name = l.assessment_id.ForeignElem.name; нет такого свойства name
 
                obj.questions = {};
 
                // --- пытаюсь достать вопросы и ответы ---
                // try
                // {
                //     alert("Открываем документ l.id = " + l.id);
 
                //     //var doc = OpenDoc(UrlFromDocID(l.id)).TopElem;
                //     //oc = OpenDoc(UrlFromDocID(l.id)).TopElem; //
                //     doc = tools.open_doc(l.id).TopElem;
 
                //     alert("doc открыт");
 
                //     var annals = tools.annals_decrypt(doc);
 
                //     alert("annals = " + tools.object_to_text(annals, 'json'));
 
                //     if (annals != null)
                //     {
                   
 
                //         alert("annals НЕ null");
                //         alert("annals.au = " + annals.au);
                //         alert("annals.au.history = " + annals.au.history);
                //         alert("annals.au.history.objects = " + annals.au.history.objects);
 
                //         //var sections = annals.au.history.objects[0].section;
                //         var history = annals.au.history;
 
 
                //         if (ArrayCount(history.objects) == 0)
                //             continue;
                //         var sections = ArrayFirstElem(history.objects).section;
 
                //         // // нормализация
                //         // if (ObjectType(sections) != 'array')
                //         //     sections = [sections];
                       
                //         alert("sections = " + sections);
 
                //         for (s in sections)
                //         {
                //             alert("section = " + s);
                //             for (q in sections[s].question)
                //             {
                //                 alert("question object = " + q);
                               
                //                 var qid = q.id;
                //                 //var qid = String(q.id);
                //                 alert("qid = " + qid);
 
                //                 if (ArrayOptFind(aQuestions, "This.id == " + qid) == undefined)
                //                 {
                //                     alert("Добавляем вопрос в aQuestions");
 
                //                     aQuestions.push({
                //                         id: qid,
                //                         text: HtmlToPlainText(q.text)
                //                     });
                //                 }
 
                //                 var qObj = new Object();
 
                //                 qObj.quest_type = (q.qtype.OptForeignElem != undefined ? q.qtype.ForeignElem.name : "");
                //                 qObj.result = tools_web.is_correct_question(q) ? "верно" : "неверно";
 
                //                 alert("variants = " + q.variant);
 
                //                 qObj.answer = ArrayMerge(q.variant, "HtmlToPlainText(value)", "; ");
                //                 qObj.correct_answer = ArrayMerge(q.variant, "HtmlToPlainText(cor_value)", "; ");
 
                //                 obj.questions[qid] = qObj;
                //             }
                //         }
                //     } else {
                //         alert("annals == NULL !!!")
                // }
                // }
                // catch(e){
                //     alert("ERROR annals block: " + e);
                // }
                // --- конец попытки ---
               
                try
{
    alert("Открываем документ l.id = " + l.id);
 
    doc = tools.open_doc(l.id).TopElem;
 
    alert("doc открыт");
 
    var annals = tools.annals_decrypt(doc);
 
    if (annals == null)
    {
        alert("annals == null");
    }
    else if (annals.au == undefined)
    {
        alert("annals.au undefined");
    }
    else if (annals.au.history == undefined)
    {
        alert("history undefined");
    }
    else if (annals.au.history.objects == undefined)
    {
        alert("objects undefined");
    }
    else if (annals.au.history.objects.object == undefined)
    {
        alert("objects.object undefined");
    }
    else
    {
        var objectsArr = annals.au.history.objects.object;
 
        alert("objectsArr count = " + ArrayCount(objectsArr));
 
        // 🔥 ВАЖНО: только for in
        for (objItem in objectsArr)
        {
            if (objItem.section == undefined)
                continue;
 
            var sections = objItem.section;
 
            alert("sections count = " + ArrayCount(sections));
 
            for (sec in sections)
            {
                if (sec.question == undefined)
                    continue;
 
                var questions = sec.question;
 
                for (q in questions)
                {
                    alert("question найден: " + q.ident);
 
                    // var qid = q.ident;
                    qid = q.ident;
 
                    if (ArrayOptFind(aQuestions, "This.id == '" + qid + "'") == undefined)
                    {
                        aQuestions.push({
                            id: qid,
                            text: q.text
                        });
                    }
 
                    qObj = new Object();
 
                    qObj.quest_type = q.qtype;
                    qObj.result = (q.state == "passed" ? "верно" : "неверно");
 
                    if (q.variant != undefined)
                        qObj.answer = ArrayMerge(q.variant, "This", "; ");
                    else
                        qObj.answer = "";
 
                    qObj.correct_answer = "";
 
                    obj.questions[qid] = qObj;
                }
            }
        }
    }
}
catch(e)
{
    alert("ERROR annals block: " + e);
}
 
                aLearnings.push(obj);
            }
 
           
        }
    }
 
    // alert("aQuestions LENGTH = " + ArrayCount(aQuestions));
    // alert("aLearnings LENGTH = " + ArrayCount(aLearnings));
 
    // var html = new Binary();
 
    // html.AppendStr("<html xmlns:o=\"urn:schemas-microsoft-com:office:office\" xmlns:x=\"urn:schemas-microsoft-com:office:excel\" xmlns=\"http://www.w3.org/TR/REC-html40\">" +
    // "<head>" +
    // "<meta http-equiv=\"Content-Type\" content=\"text/html; charset=utf-8\"/>" +
    // "<meta name=\"ProgId\" content=\"Excel.Sheet\"/>" +
    // "<meta name=\"Generator\" content=\"Microsoft Excel 11\"/>" +
    // "<style>td.two-decimals {mso-number-format: \"0\\.00\"}</style>" +
    // "</head><body><table border=\"1\" cellpadding=\"2\" cellspacing=\"0\">");
 
    // // --- Заголовок вопросов ---
    // html.AppendStr("<tr><td colspan='9'></td>");
    // for (q in aQuestions)
    // {
    //     html.AppendStr("<td colspan='4' bgcolor='#EA9999'><b>" + q.text + "</b></td>");
    // }
    // html.AppendStr("</tr>");
 
    // // --- Шапка ---
    // html.AppendStr("<tr>");
    // html.AppendStr("<td bgcolor='#FCE5CD'><b>Пользователь</b></td>");
    // html.AppendStr("<td bgcolor='#FCE5CD'><b>Название теста</b></td>");
    // html.AppendStr("<td bgcolor='#FCE5CD'><b>Код</b></td>");
    // html.AppendStr("<td bgcolor='#FCE5CD'><b>Организация</b></td>");
    // html.AppendStr("<td bgcolor='#FCE5CD'><b>Подразделение</b></td>");
    // html.AppendStr("<td bgcolor='#FCE5CD'><b>Должность</b></td>");
    // html.AppendStr("<td bgcolor='#FCE5CD'><b>Дата</b></td>");
    // html.AppendStr("<td bgcolor='#FCE5CD'><b>Статус</b></td>");
    // html.AppendStr("<td bgcolor='#FCE5CD'><b>Баллы</b></td>");
 
    // for (q in aQuestions)
    // {
    //     html.AppendStr("<td bgcolor='#FCE5CD'><b>Тип</b></td>");
    //     html.AppendStr("<td bgcolor='#FCE5CD'><b>Результат</b></td>");
    //     html.AppendStr("<td bgcolor='#FCE5CD'><b>Полученный ответ</b></td>");
    //     html.AppendStr("<td bgcolor='#FCE5CD'><b>Правильный ответ</b></td>");
    // }
    // html.AppendStr("</tr>");
 
    // //--- Данные ---
    // var K;
    // var score;
    // var percent;
    // // for (i in aLearnings)
    // for (var c = 0; c < ArrayCount(aLearnings); c++)
    // {
    //     K = aLearnings[c];
 
    //     alert("RAW_CODE = " + K.person_code);
 
    //     html.AppendStr("<tr valign='top'>");
 
    //     html.AppendStr("<td width='250'>" + K.person_fullname + "</td>");
    //     html.AppendStr("<td width='200'>" + K.assessment_name + "</td>");
    //     html.AppendStr("<td width='100'>" + K.person_code + "</td>");
    //     html.AppendStr("<td width='150'>" + K.org + "</td>");
    //     html.AppendStr("<td width='150'>" + K.subdivision + "</td>");
    //     html.AppendStr("<td width='150'>" + K.position + "</td>");  
    //     html.AppendStr("<td width='110' align='center'>" + StrDate(K.date, false, false) + "</td>");
    //     //html.AppendStr("<td width='110' align='center'>" + K.date + "</td>");
    //     html.AppendStr("<td width='100' align='center'>" + K.status + "</td>");
 
    //     score = "";
    //     if (K.max_score != null && K.max_score != 0)
    //     {
    //         percent = Math.round(100 * K.score / K.max_score);
    //         score = K.score + " (" + percent + "%)";
    //     }
    //     else
    //     {
    //         score = K.score;
    //     }
 
    //     html.AppendStr("<td width='100' align='center'>" + score + "</td>");
 
    //     for (q in aQuestions)
    //     {
    //         //var qid = aQuestions[q].id;
    //         qid = q.id;
    //         qData = K.questions[qid];
           
 
    //         if (qData == undefined)
    //         {
    //             html.AppendStr("<td></td><td></td><td></td><td></td>");
    //         }
    //         else
    //         {
    //             var bg = qData.result == "неверно" ? "#F4CCCC" : "#D9EAD3";
 
    //             html.AppendStr("<td width='150'>" + qData.quest_type + "</td>");
    //             html.AppendStr("<td width='80' align='center' bgcolor='" + bg + "'>" + qData.result + "</td>");
    //             html.AppendStr("<td width='250'>" + qData.answer + "</td>");
    //             html.AppendStr("<td width='250'>" + qData.correct_answer + "</td>");
    //         }
    //     }
 
    //     html.AppendStr("</tr>");
    // }
 
    // html.AppendStr("</table></body></html>");
 
    // alert("конец функции createReportString");

    alert("aQuestions LENGTH = " + ArrayCount(aQuestions));
    alert("aLearnings LENGTH = " + ArrayCount(aLearnings));

    var html = new Binary();

html.AppendStr("<html xmlns:o=\"urn:schemas-microsoft-com:office:office\" xmlns:x=\"urn:schemas-microsoft-com:office:excel\" xmlns=\"http://www.w3.org/TR/REC-html40\">" +
"<head>" +
"<meta http-equiv=\"Content-Type\" content=\"text/html; charset=utf-8\"/>" +
"<meta name=\"ProgId\" content=\"Excel.Sheet\"/>" +
"<meta name=\"Generator\" content=\"Microsoft Excel 11\"/>" +
"<style>td.two-decimals {mso-number-format: \"0\\.00\"}</style>" +
"</head><body><table border=\"1\" cellpadding=\"2\" cellspacing=\"0\">");

// --- Заголовок вопросов ---
html.AppendStr("<tr><td colspan='9'></td>");
for (q in aQuestions)
{
    html.AppendStr("<td colspan='4' bgcolor='#EA9999'><b>" + q.text + "</b></td>");
}
html.AppendStr("</tr>");

// --- Шапка ---
html.AppendStr("<tr>");
html.AppendStr("<td bgcolor='#FCE5CD'><b>Пользователь</b></td>");
html.AppendStr("<td bgcolor='#FCE5CD'><b>Название теста</b></td>");
html.AppendStr("<td bgcolor='#FCE5CD'><b>Код</b></td>");
html.AppendStr("<td bgcolor='#FCE5CD'><b>Организация</b></td>");
html.AppendStr("<td bgcolor='#FCE5CD'><b>Подразделение</b></td>");
html.AppendStr("<td bgcolor='#FCE5CD'><b>Должность</b></td>");
html.AppendStr("<td bgcolor='#FCE5CD'><b>Дата</b></td>");
html.AppendStr("<td bgcolor='#FCE5CD'><b>Статус</b></td>");
html.AppendStr("<td bgcolor='#FCE5CD'><b>Баллы</b></td>");

for (q in aQuestions)
{
    html.AppendStr("<td bgcolor='#FCE5CD'><b>Тип</b></td>");
    html.AppendStr("<td bgcolor='#FCE5CD'><b>Результат</b></td>");
    html.AppendStr("<td bgcolor='#FCE5CD'><b>Полученный ответ</b></td>");
    html.AppendStr("<td bgcolor='#FCE5CD'><b>Правильный ответ</b></td>");
}
html.AppendStr("</tr>");

// --- Данные ---
  var K;
  var score;
  var percent;
  
for (var c = 0; c < ArrayCount(aLearnings); c++)
{
     K = aLearnings[c];

    html.AppendStr("<tr valign='top'>");

    html.AppendStr("<td width='250'>" + K.person_fullname + "</td>");
    html.AppendStr("<td width='200'>" + K.assessment_name + "</td>");
    html.AppendStr("<td width='100'>" + K.person_code + "</td>");
    html.AppendStr("<td width='150'>" + K.org + "</td>");
    html.AppendStr("<td width='150'>" + K.subdivision + "</td>");
    html.AppendStr("<td width='150'>" + K.position + "</td>");
    html.AppendStr("<td width='110' align='center'>" + StrDate(K.date, false, false) + "</td>");
    html.AppendStr("<td width='100' align='center'>" + K.status + "</td>");

     score = "";
    if (K.max_score != null && K.max_score != 0)
    {
        percent = Math.round(100 * K.score / K.max_score);
        score = K.score + " (" + percent + "%)";
    }
    else
    {
        score = K.score;
    }

    html.AppendStr("<td width='100' align='center'>" + score + "</td>");

    for (q in aQuestions)
    {
         qid = q.id;
         qData = K.questions[qid];

        if (qData == undefined)
        {
            html.AppendStr("<td></td><td></td><td></td><td></td>");
        }
        else
        {
             bg = qData.result == "неверно" ? "#F4CCCC" : "#D9EAD3";

            html.AppendStr("<td width='150'>" + qData.quest_type + "</td>");
            html.AppendStr("<td width='80' align='center' bgcolor='" + bg + "'>" + qData.result + "</td>");
            html.AppendStr("<td width='250'>" + qData.answer + "</td>");
            html.AppendStr("<td width='250'>" + qData.correct_answer + "</td>");
        }
    }

    html.AppendStr("</tr>");
}

html.AppendStr("</table></body></html>");
 
    alert("конец функции createReportString");
    return html.GetStr();
   
}
 
// --- Загружаем данные формы ---
aFormFields = ParseJson(getParam("form_fields", "[]"));
aFormFieldsDef = ParseJson(getParam("form_fields_default", "[]"));
sSubmitType = getFormField("__submit_type__", getFormFieldDefault("__submit_type__", "main_data"));
 
// --- Форма ---
oForm = new Object();
oForm.command = "display_form";
oForm.height = 320;
oForm.title = 'Выгрузка отчета по обучению';
oForm.message = 'Здесь вы можете выгрузить отчет по результатам прохождения теста сотрудником';
oForm.form_fields = [
    {
        name: "select_collaborator_ids",
        label: "Выберите сотрудников, по которыми хотите выгрузить отчет",
        title: "Выберите сотрудников",
        type: "foreign_elem",
        value: "",//ArrayMerge(oSelect_collaborator_ids, "This", ";"),
        mandatory: false,
        multiple: false,
        catalog: "collaborator",
        query_qual: ""//("MatchSome($elem/id,(" + ArrayMerge(queryCollaborator_ids, "This", ",") + "))")
    },
    {
        name: "select_assessment_ids",
        title: "Выберите тесты",
        label: "Выберите тесты, по которыми хотите выгрузить отчет",
        type: "foreign_elem",
        value: "",//ArrayMerge(oSelect_assessment_ids, "This", ";"),
        mandatory: false,
        multiple: true,
        catalog: "assessment",
        query_qual: ""//("$elem/status='publish'")
    },{
        name: "startDate",         //f_final_date  
        label: "Выгрузить прохождения ОТ",
        type: "date",
        value: '',
        //value: StrDate(Date('01.01.1970'), true, false),
        //value: StrDate(Date('01.01.2002'), false, false),
        mandatory: true,
        multiple: false,
        query_qual: "",
        page: "page_one"        
    },{
        name: "finalDate",         //f_final_date  
        label: "Выгрузить прохождения ДО",
        type: "date",
        value: '',
       // value: StrDate(Date(), true, false),//getFormField("finalDate", oParentDocTE.custom_elems.ObtainChildByKey("f_final_date").value),
        //value: StrDate(Date(), false, false),
        mandatory: true,
        multiple: false,
        query_qual: ""      
    }
];


// --- Кнопки и логика шагов ---
oForm.buttons = [];
oForm.no_buttons = false;
 
switch(sSubmitType){
    case "main_data": {
        //aMandatoryFields = ["name", "sheCode", "function", "vacancyReason", "workCity", "workPlace"];
        aMandatoryFields = ["select_collaborator_ids", "select_assessment_ids"];
        aVisibleFields = ["select_collaborator_ids", "select_assessment_ids", "startDate", "finalDate"];
        for (oField in oForm.form_fields) {
            oField.type = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "hidden" : oField.type );
            try {
                oField.validation = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "" : oField.validation );
            }
            catch(_e){}
           
            oField.mandatory = (ArrayOptFind(aMandatoryFields, "This == oField.name") != undefined);
            //oField.value = getFormField(oField.name, oField.value);
 
            if (oField.name == "subdivision_id" || oField.name == "subdivision_text") {
                oField.value = getFormField(oField.name, "");
            } else {
                oField.value = getFormField(oField.name, oField.value);
            }
        }
        oForm.buttons.push({name:"submit", submit_type:"create_report", label:"Создать отчет", type:"submit"},{name:"cancel", label:"Отменить", type:"cancel"});
        RESULT = oForm;
        break;
    }
    case "create_report": {
       
 
        var oSelect_collaborator_ids = getFormField('select_collaborator_ids', "" );
        //alert("oSelect_collaborator_ids " + oSelect_collaborator_ids);
 
        //if ( oSelect_collaborator_ids != undefined && oSelect_collaborator_ids.value != "" ) { // --- тут падает ---
        if ( oSelect_collaborator_ids != undefined && oSelect_collaborator_ids != "" ) {
          //  oSelect_collaborator_ids = String( oSelect_collaborator_ids.value ).split( ";" );
            oSelect_collaborator_ids = String( oSelect_collaborator_ids ).split( ";" );
        }
        else {
            oSelect_collaborator_ids = [];
        }
 
        var oSelect_assessment_ids = getFormField('select_assessment_ids', "" );
        //alert("oSelect_assessment_ids " + oSelect_assessment_ids);
 
        //if ( oSelect_assessment_ids != undefined && oSelect_assessment_ids.value != "" ) {
        if ( oSelect_assessment_ids != undefined && oSelect_assessment_ids != "" ) {
             //oSelect_assessment_ids = String( oSelect_assessment_ids.value ).split( ";" );
            oSelect_assessment_ids = String( oSelect_assessment_ids ).split( ";" );
        }
        else {
            oSelect_assessment_ids = [];
        }
 
        var startDate = getFormField('startDate', '');
        // var startDate = getFormField('startDate', '01.01.2002');
        startDate = ParseDate(startDate);
        var endDate = getFormField('finalDate', StrDate(Date(), false));
        endDate = ParseDate(endDate);
        // alert(startDate);
        // alert(endDate);
 
        // ---
        // alert("collab RAW = " + oSelect_collaborator_ids);
        // alert("assessment RAW = " + oSelect_assessment_ids);
        // ---
       
        sReportStr = createReportString(oSelect_collaborator_ids, oSelect_assessment_ids, startDate, endDate);
        // ---
        // alert("collab AFTER SPLIT = " + ArrayMerge(oSelect_collaborator_ids, "This", "|"));
        // alert("assessment AFTER SPLIT = " + ArrayMerge(oSelect_assessment_ids, "This", "|"));
        // ---
        //alert("sReportString = " + sReportStr);
 
        tools_web.set_user_data("excel_html_" + curUserID, ({
            "html" : sReportStr
        }), 3600);
        oForm = {
            command: "new_window",
            url: "assessment_excel_export.html"
        }
        break;
    }
   
    default: {
        oForm = {
            command: "alert",
            msg: ("Нет выбранного действия"),
            view: "warning",
            title: "Ошибки заполнения"
        };
        break;
    }
}
 
RESULT = oForm;
