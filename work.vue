function createReportString(aCollaboratorIDArray, aAssessmentIDArray, dStartDate, dFinishDate)
{
    alert("старт функции createReportString");

    var aLearnings = [];
    var aQuestions = [];

    for (var i = 0; i < ArrayCount(aCollaboratorIDArray); i++)
    {
        var raw_person_id = aCollaboratorIDArray[i];

        if (raw_person_id == undefined)
            continue;

        raw_person_id = Trim(raw_person_id);

        if (raw_person_id == "")
            continue;

        var person_id = raw_person_id;

        for (var j = 0; j < ArrayCount(aAssessmentIDArray); j++)
        {
            var raw_ass_id = aAssessmentIDArray[j];

            if (raw_ass_id == undefined)
                continue;

            raw_ass_id = Trim(raw_ass_id);

            if (raw_ass_id == "")
                continue;

            var ass_id = raw_ass_id;

            var arr = XQuery(
                "for $l in test_learnings " +
                "where $l/person_id = " + XQueryLiteral(person_id) +
                " and $l/assessment_id = " + XQueryLiteral(ass_id) +
                " and $l/start_usage_date >= date('" + StrDate(dStartDate,false) + "')" +
                " and $l/start_usage_date <= date('" + StrDate(dFinishDate,false) + "')" +
                " return $l"
            );

            for (l in arr)
            {
                var obj = new Object();

                obj.person_fullname = l.person_id.ForeignElem.fullname;
                obj.assessment_name = l.assessment_name;
                obj.person_code = l.person_id.ForeignElem.code;
                obj.org = l.person_id.ForeignElem.org_name;
                obj.subdivision = l.person_id.ForeignElem.position_parent_name;
                obj.position = l.person_id.ForeignElem.position_name;
                obj.date = l.start_usage_date;
                obj.status = l.state_id.ForeignElem.name;
                obj.score = l.score;
                obj.max_score = l.max_score;

                obj.questions = {};

                try
                {
                    var doc = tools.open_doc(l.id).TopElem;
                    var annals = tools.annals_decrypt(doc);

                    if (
                        annals != null &&
                        annals.au != undefined &&
                        annals.au.history != undefined &&
                        annals.au.history.objects != undefined &&
                        annals.au.history.objects.object != undefined
                    )
                    {
                        var objectsArr = annals.au.history.objects.object;

                        for (objItem in objectsArr)
                        {
                            if (objItem.section == undefined)
                                continue;

                            var sections = objItem.section;

                            for (sec in sections)
                            {
                                if (sec.question == undefined)
                                    continue;

                                var questions = sec.question;

                                for (q in questions)
                                {
                                    var qid = String(q.ident); // FIX

                                    if (ArrayOptFind(aQuestions, "This.id == '" + qid + "'") == undefined)
                                    {
                                        aQuestions.push({
                                            id: qid, // FIX
                                            text: q.text
                                        });
                                    }

                                    var qObj = new Object();

                                    // тип
                                    if (q.qtype != undefined && q.qtype.OptForeignElem != undefined)
                                        qObj.quest_type = q.qtype.ForeignElem.name;
                                    else
                                        qObj.quest_type = q.qtype;

                                    // результат
                                    try
                                    {
                                        qObj.result = tools_web.is_correct_question(q) ? "верно" : "неверно";
                                    }
                                    catch(e)
                                    {
                                        qObj.result = "";
                                    }

                                    // нормализация variant
                                    var variants = [];

                                    if (q.variant != undefined)
                                    {
                                        if (ObjectType(q.variant) == "array")
                                            variants = q.variant;
                                        else
                                            variants = [q.variant];
                                    }

                                    // правильный ответ
                                    switch (q.qtype)
                                    {
                                        case "choice":
                                        case "select":
                                            qObj.correct_answer = ArrayMerge(
                                                ArraySelect(variants, "correct == '1'"),
                                                "HtmlToPlainText(text)",
                                                "; "
                                            );
                                            break;

                                        default:
                                            qObj.correct_answer = ArrayMerge(
                                                variants,
                                                "HtmlToPlainText(cor_value)",
                                                "; "
                                            );
                                            break;
                                    }

                                    // ответ пользователя
                                    switch (q.qtype)
                                    {
                                        case "choice":
                                        case "select":
                                            qObj.answer = ArrayMerge(
                                                ArraySelect(variants, "Trim(This) == '1'"),
                                                "HtmlToPlainText(text)",
                                                "; "
                                            );
                                            break;

                                        default:
                                            qObj.answer = ArrayMerge(
                                                variants,
                                                "HtmlToPlainText(value)",
                                                "; "
                                            );
                                            break;
                                    }

                                    obj.questions[qid] = qObj; // оставляем твою логику
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

    var html = new Binary();

    html.AppendStr("<html xmlns:o=\"urn:schemas-microsoft-com:office:office\" xmlns:x=\"urn:schemas-microsoft-com:office:excel\" xmlns=\"http://www.w3.org/TR/REC-html40\">" +
        "<head>" +
        "<meta http-equiv=\"Content-Type\" content=\"text/html; charset=utf-8\"/>" +
        "<meta name=\"ProgId\" content=\"Excel.Sheet\"/>" +
        "<meta name=\"Generator\" content=\"Microsoft Excel 11\"/>" +
        "<style>td.two-decimals {mso-number-format: \"0\\.00\"}</style>" +
        "</head><body><table border=\"1\" cellpadding=\"2\" cellspacing=\"0\">");

    html.AppendStr("<tr><td colspan='9'></td>");
    for (qh in aQuestions)
        html.AppendStr("<td colspan='4' bgcolor='#EA9999'><b>" + qh.text + "</b></td>");
    html.AppendStr("</tr>");

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

    for (var c = 0; c < ArrayCount(aLearnings); c++)
    {
        var K = aLearnings[c];

        html.AppendStr("<tr>");

        html.AppendStr("<td width='250'>" + K.person_fullname + "</td>");
        html.AppendStr("<td width='200'>" + K.assessment_name + "</td>");
        html.AppendStr("<td width='100'>" + K.person_code + "</td>");
        html.AppendStr("<td width='150'>" + K.org + "</td>");
        html.AppendStr("<td width='150'>" + K.subdivision + "</td>");
        html.AppendStr("<td width='150'>" + K.position + "</td>");
        html.AppendStr("<td width='150' align='center'>" + StrDate(K.date, true, false) + "</td>");
        html.AppendStr("<td width='100' align='center'>" + K.status + "</td>");

        var score = "";
        if (K.max_score != null && K.max_score != 0)
            score = K.score + " (" + Math.round(100 * K.score / K.max_score) + "%)";
        else
            score = K.score;

        html.AppendStr("<td width='100' align='center'>" + score + "</td>");

        for (var qh3 = 0; qh3 < ArrayCount(aQuestions); qh3++)
        {
            var qid2 = String(aQuestions[qh3].id); // FIX

            var qData;

            if (K.questions.HasProperty(qid2)) // FIX
                qData = K.questions[qid2];
            else
                qData = undefined;

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
                  
