
 
function createReportString(aCollaboratorIDArray, aAssessmentIDArray, dStartDate, dFinishDate)
{
    alert("старт функции createReportString");
 
   
    var aLearnings = [];
    var aQuestions = [];
 
    // for (i in aCollaboratorIDArray)
    for (var i = 0; i < ArrayCount(aCollaboratorIDArray); i++) {
        //var person_id = OptInt(aCollaboratorIDArray[i]);
 
        var raw_person_id = aCollaboratorIDArray[i];
        //alert("raw_person_id = " + raw_person_id);
 
        if (raw_person_id == undefined)
            continue;
 
        raw_person_id = Trim(raw_person_id);
 
        if (raw_person_id == "")
            continue;
 
        var person_id = raw_person_id;
        alert("person_id = " + person_id);
       
 
        //for (j in aAssessmentIDArray)
        for (var j = 0; j < ArrayCount(aAssessmentIDArray); j++)
        {
            //var ass_id = OptInt(aAssessmentIDArray[j]);
            alert("Массив IDs assesments " + aAssessmentIDArray);
 
            var raw_ass_id = aAssessmentIDArray[j];
            //alert("ass_id = " + raw_ass_id);
 
            if (raw_ass_id == undefined)
                continue;
 
            raw_ass_id = Trim(raw_ass_id);
 
            if (raw_ass_id == "")
                continue;
 
            var ass_id = raw_ass_id;
            alert("ass_id = " + ass_id);
 
           
           
            var arr = XQuery(
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


             for (l in arr)
            // for (var k = 0; k < ArrayCount(items); k++)
            {
                // var l = arr[k]; // тут падает
 
                var obj = new Object();
 
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
 
                try
                {
                    var doc = OpenDoc(UrlFromDocID(l.id)).TopElem;
                    var annals = tools.annals_decrypt(doc);
 
                    if (annals != null)
                    {
                        var sections = annals.au.history.objects[0].section;
 
                        for (s in sections)
                        {
                            for (q in sections[s].question)
                            {
                                var qid = q.id;
 
                                if (ArrayOptFind(aQuestions, "This.id == " + qid) == undefined)
                                {
                                    aQuestions.push({
                                        id: qid,
                                        text: HtmlToPlainText(q.text)
                                    });
                                }
 
                                var qObj = new Object();
 
                                qObj.quest_type = (q.qtype.OptForeignElem != undefined ? q.qtype.ForeignElem.name : "");
                                qObj.result = tools_web.is_correct_question(q) ? "верно" : "неверно";
                                qObj.answer = ArrayMerge(q.variant, "HtmlToPlainText(value)", "; ");
                                qObj.correct_answer = ArrayMerge(q.variant, "HtmlToPlainText(cor_value)", "; ");
 
                                obj.questions[qid] = qObj;
                            }
                        }
                    }
                }
                catch(e){}
 
                aLearnings.push(obj);
            }
        }
    }
 
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
        html.AppendStr("<td colspan='4' bgcolor='#EA9999'><b>" + aQuestions[q].text + "</b></td>");
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
 
    //--- Данные ---
    // for (i in aLearnings)
    for (var c = 0; c < ArrayCount(aLearnings); c++)
    {
        var K = aLearnings[c];
 
        alert("RAW_CODE = " + K.person_code);
 
        html.AppendStr("<tr valign='top'>");
 
        html.AppendStr("<td width='250'>" + K.person_fullname + "</td>");
        html.AppendStr("<td width='200'>" + K.assessment_name + "</td>"); //такого не находит
        html.AppendStr("<td width='100'>" + K.person_code + "</td>");
        html.AppendStr("<td width='150'>" + K.org + "</td>");
        html.AppendStr("<td width='150'>" + K.subdivision + "</td>");
        html.AppendStr("<td width='150'>" + K.position + "</td>");  
        html.AppendStr("<td width='110' align='center'>" + StrDate(K.date, false, false) + "</td>");
        //html.AppendStr("<td width='110' align='center'>" + K.date + "</td>");
        html.AppendStr("<td width='100' align='center'>" + K.status + "</td>");
 
        var score = "";
        if (K.max_score != null && K.max_score != 0)
        {
            var percent = Math.round(100 * K.score / K.max_score);
            score = K.score + " (" + percent + "%)";
        }
        else
        {
            score = K.score;
        }
 
        html.AppendStr("<td width='100' align='center'>" + score + "</td>");
 
        for (q in aQuestions)
        {
            var qid = aQuestions[q].id;
            var qData = K.questions[qid];
 
            if (qData == undefined)
            {
                html.AppendStr("<td></td><td></td><td></td><td></td>");
            }
            else
            {
                var bg = qData.result == "неверно" ? "#F4CCCC" : "#D9EAD3";
 
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
 
RESULT = oForm;
