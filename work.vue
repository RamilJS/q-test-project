function createReportString(aCollaboratorIDArray, aAssessmentIDArray, dStartDate, dFinishDate)
{
    alert("старт функции createReportString");

    var aLearnings = [];
    var aQuestions = [];

    for (var i = 0; i < ArrayCount(aCollaboratorIDArray); i++)
    {
        var person_id = Trim(aCollaboratorIDArray[i]);
        if (person_id == "")
            continue;

        alert("person_id = " + person_id);

        // --- формируем условие по тестам ---
        var ass_cond = "";

        for (var j = 0; j < ArrayCount(aAssessmentIDArray); j++)
        {
            var ass_id = Trim(aAssessmentIDArray[j]);
            if (ass_id == "")
                continue;

            if (ass_cond != "")
                ass_cond += " or ";

            ass_cond += "$l/assessment_id = " + XQueryLiteral(ass_id);
        }

        if (ass_cond == "")
            continue;

        // --- XQuery ---
        var query =
            "for $l in test_learnings " +
            "where $l/person_id = " + XQueryLiteral(person_id) +
            " and (" + ass_cond + ")" +
            " and $l/start_usage_date >= date('" + StrDate(dStartDate,false) + "')" +
            " and $l/start_usage_date <= date('" + StrDate(dFinishDate,false) + "')" +
            " return $l";

        var arr = XQuery(query);

        // --- приводим к массиву ---
        if (arr == undefined)
            continue;

        if (!ArrayIsArray(arr))
            arr = [arr];

        alert("найдено записей = " + ArrayCount(arr));

        // --- обработка ---
        for (var k = 0; k < ArrayCount(arr); k++)
        {
            var l = arr[k];

            var obj = new Object();

            obj.person_fullname = l.person_id.ForeignElem.fullname;
            obj.person_code = l.person_id.ForeignElem.code;
            obj.org = l.person_id.ForeignElem.org_name;
            obj.subdivision = l.person_id.ForeignElem.position_parent_name;
            obj.position = l.person_id.ForeignElem.position_name;
            obj.date = l.start_usage_date;
            obj.status = l.state_id.ForeignElem.name;
            obj.score = l.score;
            obj.max_score = l.max_score;
            obj.assessment_name = l.assessment_id.ForeignElem.name;

            obj.questions = {};

            try
            {
                var doc = OpenDoc(UrlFromDocID(l.id)).TopElem;
                var annals = tools.annals_decrypt(doc);

                if (annals != null && ArrayCount(annals.au.history.objects) > 0)
                {
                    var sections = annals.au.history.objects[0].section;

                    for (var s = 0; s < ArrayCount(sections); s++)
                    {
                        var questions = sections[s].question;

                        for (var q = 0; q < ArrayCount(questions); q++)
                        {
                            var quest = questions[q];
                            var qid = quest.id;

                            // --- список всех вопросов ---
                            if (ArrayOptFind(aQuestions, "This.id == " + qid) == undefined)
                            {
                                var qText = HtmlToPlainText(quest.text);

                                aQuestions.push({
                                    id: qid,
                                    text: qText
                                });
                            }

                            var qObj = new Object();

                            qObj.quest_type = (quest.qtype.OptForeignElem != undefined ? quest.qtype.ForeignElem.name : "");
                            qObj.result = tools_web.is_correct_question(quest) ? "верно" : "неверно";

                            // ответы
                            qObj.answer = ArrayMerge(quest.variant, "HtmlToPlainText(value)", "; ");
                            qObj.correct_answer = ArrayMerge(quest.variant, "HtmlToPlainText(cor_value)", "; ");

                            obj.questions[qid] = qObj;
                        }
                    }
                }
            }
            catch(e)
            {
                alert("ошибка обработки вопросов: " + e);
            }

            aLearnings.push(obj);
        }
    }

    alert("всего записей = " + ArrayCount(aLearnings));

    // --- HTML ---
    var html = new Binary();

    html.AppendStr("<html><head><meta http-equiv='Content-Type' content='text/html; charset=utf-8'/></head><body>");
    html.AppendStr("<table border='1'>");

    // --- заголовок вопросов ---
    html.AppendStr("<tr><td colspan='9'></td>");
    for (var q = 0; q < ArrayCount(aQuestions); q++)
    {
        html.AppendStr("<td colspan='4'><b>" + aQuestions[q].text + "</b></td>");
    }
    html.AppendStr("</tr>");

    // --- шапка ---
    html.AppendStr("<tr>");
    html.AppendStr("<td><b>Пользователь</b></td>");
    html.AppendStr("<td><b>Тест</b></td>");
    html.AppendStr("<td><b>Код</b></td>");
    html.AppendStr("<td><b>Организация</b></td>");
    html.AppendStr("<td><b>Подразделение</b></td>");
    html.AppendStr("<td><b>Должность</b></td>");
    html.AppendStr("<td><b>Дата</b></td>");
    html.AppendStr("<td><b>Статус</b></td>");
    html.AppendStr("<td><b>Баллы</b></td>");

    for (var q = 0; q < ArrayCount(aQuestions); q++)
    {
        html.AppendStr("<td>Тип</td>");
        html.AppendStr("<td>Результат</td>");
        html.AppendStr("<td>Ответ</td>");
        html.AppendStr("<td>Правильный</td>");
    }

    html.AppendStr("</tr>");

    // --- данные ---
    for (var i = 0; i < ArrayCount(aLearnings); i++)
    {
        var l = aLearnings[i];

        html.AppendStr("<tr>");

        html.AppendStr("<td>" + l.person_fullname + "</td>");
        html.AppendStr("<td>" + l.assessment_name + "</td>");
        html.AppendStr("<td>" + l.person_code + "</td>");
        html.AppendStr("<td>" + l.org + "</td>");
        html.AppendStr("<td>" + l.subdivision + "</td>");
        html.AppendStr("<td>" + l.position + "</td>");
        html.AppendStr("<td>" + StrDate(l.date, true, false) + "</td>");
        html.AppendStr("<td>" + l.status + "</td>");

        var score = l.score;
        if (l.max_score != null && l.max_score != 0)
        {
            var percent = Math.round(100 * l.score / l.max_score);
            score = l.score + " (" + percent + "%)";
        }

        html.AppendStr("<td>" + score + "</td>");

        for (var q = 0; q < ArrayCount(aQuestions); q++)
        {
            var qid = aQuestions[q].id;
            var qData = l.questions[qid];

            if (qData == undefined)
            {
                html.AppendStr("<td></td><td></td><td></td><td></td>");
            }
            else
            {
                html.AppendStr("<td>" + qData.quest_type + "</td>");
                html.AppendStr("<td>" + qData.result + "</td>");
                html.AppendStr("<td>" + qData.answer + "</td>");
                html.AppendStr("<td>" + qData.correct_answer + "</td>");
            }
        }

        html.AppendStr("</tr>");
    }

    html.AppendStr("</table></body></html>");

    alert("конец функции createReportString");

    return html.GetStr();
}


DeepSeek
