function createReportString(aCollaboratorIDArray, aAssessmentIDArray, dStartDate, dFinishDate)
{
    alert("старт функции createReportString");

    var aLearnings = [];
    var aQuestions = [];

    var raw_person_id;

    for (var i = 0; i < ArrayCount(aCollaboratorIDArray); i++) {

        raw_person_id = aCollaboratorIDArray[i];

        if (raw_person_id == undefined)
            continue;

        raw_person_id = Trim(raw_person_id);

        if (raw_person_id == "")
            continue;

        var person_id = raw_person_id;
        alert("person_id = " + person_id);

        for (var j = 0; j < ArrayCount(aAssessmentIDArray); j++)
        {
            alert("Массив IDs assesments " + aAssessmentIDArray);

            var raw_ass_id = aAssessmentIDArray[j];

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

            alert("ArrayCount(arr) = " + ArrayCount(arr));

            for (l in arr)
            {
                obj = new Object();

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

                // --- FIX: теперь массив ---
                obj.questions = [];

                try
                {
                    doc = tools.open_doc(l.id).TopElem;

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

                        for (var oi = 0; oi < ArrayCount(objectsArr); oi++)
                        {
                            var objItem = objectsArr[oi];

                            if (objItem.section == undefined)
                                continue;

                            var sections = objItem.section;

                            if (ObjectType(sections) != "array")
                                sections = [sections];

                            for (var si = 0; si < ArrayCount(sections); si++)
                            {
                                var sec = sections[si];

                                if (sec.question == undefined)
                                    continue;

                                var questions = sec.question;

                                if (ObjectType(questions) != "array")
                                    questions = [questions];

                                for (var qi = 0; qi < ArrayCount(questions); qi++)
                                {
                                    var q = questions[qi];
                                    var qid = String(q.ident);

                                    if (ArrayOptFind(aQuestions, "This.id == '" + qid + "'") == undefined)
                                    {
                                        aQuestions.push({
                                            id: qid,
                                            text: HtmlToPlainText(q.text)
                                        });
                                    }

                                    var qObj = new Object();
                                    qObj.id = qid;

                                    // тип
                                    qObj.quest_type = (q.qtype.OptForeignElem != undefined)
                                        ? q.qtype.ForeignElem.name
                                        : q.qtype;

                                    // результат
                                    try
                                    {
                                        qObj.result = tools_web.is_correct_question(q) ? "верно" : "неверно";
                                    }
                                    catch(e)
                                    {
                                        qObj.result = "";
                                    }

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

                                        case "range":
                                            var tmp = "";
                                            for (var ii = 0; ii < ArrayCount(variants); ii++)
                                                tmp += (tmp == "" ? "" : "; ") + ii;
                                            qObj.correct_answer = tmp;
                                            break;

                                        case "numeric":
                                            qObj.correct_answer = ArrayMerge(
                                                variants,
                                                "ArrayMerge(cond,'operator.ForeignElem.name + Trim(Value)',', ')",
                                                "; "
                                            );
                                            break;

                                        case "text":
                                            qObj.correct_answer = ArrayMerge(
                                                variants,
                                                "ArrayMerge(cond,'operator==\\'cn\\'?\\'...\\'+Trim(Value)+\\'...\\':Trim(Value)',', ')",
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

                                        case "range":
                                            qObj.answer = ArrayMerge(
                                                ArraySelect(variants, "Trim(value) != ''"),
                                                "HtmlToPlainText(value)",
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

                                    obj.questions.push(qObj);
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

    html.AppendStr("<tr><td colspan='9'></td>");
    for (qh in aQuestions)
    {
        html.AppendStr("<td colspan='4' bgcolor='#EA9999'><b>" + qh.text + "</b></td>");
    }
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

    var K, score;

    for (var c = 0; c < ArrayCount(aLearnings); c++)
    {
        K = aLearnings[c];

        html.AppendStr("<tr>");

        html.AppendStr("<td width='250'>" + K.person_fullname + "</td>");
        html.AppendStr("<td width='200'>" + K.assessment_name + "</td>");
        html.AppendStr("<td width='100'>" + K.person_code + "</td>");
        html.AppendStr("<td width='150'>" + K.org + "</td>");
        html.AppendStr("<td width='150'>" + K.subdivision + "</td>");
        html.AppendStr("<td width='150'>" + K.position + "</td>");
        html.AppendStr("<td width='150' align='center'>" + StrDate(K.date, true, false) + "</td>");
        html.AppendStr("<td width='100' align='center'>" + K.status + "</td>");

        if (K.max_score != null && K.max_score != 0)
            score = K.score + " (" + Math.round(100 * K.score / K.max_score) + "%)";
        else
            score = K.score;

        html.AppendStr("<td width='100' align='center'>" + score + "</td>");

        for (var qh3 = 0; qh3 < ArrayCount(aQuestions); qh3++)
        {
            var qid2 = aQuestions[qh3].id;

            // --- FIX ---
            var qData = ArrayOptFind(K.questions, "This.id == '" + qid2 + "'");

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
