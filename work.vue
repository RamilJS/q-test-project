
function createReportString(aCollaboratorIDArray, aAssessmentIDArray, dStartDate, dFinishDate)
{
    alert("старт функции createReportString");
    
    var aLearnings = [];
    var aQuestions = [];

    for (var i = 0; i < ArrayCount(aCollaboratorIDArray); i++) {
        var raw_person_id = aCollaboratorIDArray[i];
        if (raw_person_id == undefined || Trim(raw_person_id) == "") continue;
        var person_id = raw_person_id;
        
        for (var j = 0; j < ArrayCount(aAssessmentIDArray); j++) {
            var raw_ass_id = aAssessmentIDArray[j];
            if (raw_ass_id == undefined || Trim(raw_ass_id) == "") continue;
            var ass_id = raw_ass_id;
            
            var arr = XQuery(
                "for $l in test_learnings " +
                "where $l/person_id = " + XQueryLiteral(person_id) +
                " and $l/assessment_id = " + XQueryLiteral(ass_id) +
                " and $l/start_usage_date >= date('" + StrDate(dStartDate,false) + "')" +
                " and $l/start_usage_date <= date('" + StrDate(dFinishDate,false) + "')" +
                " return $l"
            );
            
            for (var k = 0; k < ArrayCount(arr); k++) {
                var learning = arr[k];
                
                // Базовый объект
                var obj = {
                    person_fullname: learning.person_fullname,
                    assessment_name: learning.assessment_name,
                    person_code: (learning.person_id && learning.person_id.ForeignElem) ? learning.person_id.ForeignElem.code : "",
                    org: learning.person_org_name,
                    subdivision: learning.person_subdivision_name,
                    position: learning.person_position_name,
                    date: learning.start_usage_date,
                    status: (learning.state_id && learning.state_id.ForeignElem) ? learning.state_id.ForeignElem.name : learning.state_id,
                    score: learning.score,
                    max_score: learning.max_score,
                    questions: {}
                };
                
                // Пытаемся получить анналы
                try {
                    var doc = OpenDoc(UrlFromDocID(learning.id)).TopElem;
                    var annals = tools.annals_decrypt(doc);
                    
                    if (annals && annals.au && annals.au.history && ArrayCount(annals.au.history.objects) > 0) {
                        var sections = annals.au.history.objects[0].section;
                        if (sections) {
                            for (var s = 0; s < ArrayCount(sections); s++) {
                                var questions = sections[s].question;
                                if (!questions) continue;
                                for (var q = 0; q < ArrayCount(questions); q++) {
                                    var question = questions[q];
                                    var qid = question.PrimaryKey ? String(question.PrimaryKey) : String(question.id);
                                    
                                    // Добавляем вопрос в общий список
                                    if (ArrayOptFind(aQuestions, "This.id == '" + qid + "'") == undefined) {
                                        aQuestions.push({
                                            id: qid,
                                            text: HtmlToPlainText(question.text || "")
                                        });
                                    }
                                    
                                    // Данные ответа
                                    var qObj = {};
                                    qObj.quest_type = (question.qtype && question.qtype.ForeignElem) ? question.qtype.ForeignElem.name : (question.qtype || "");
                                    var hasWeight = (ArrayOptFind(question.variant, "This.varscore.HasValue") != undefined);
                                    qObj.result = (!hasWeight && tools_web.is_correct_question(question)) ? "верно" : "неверно";
                                    
                                    // Ответы
                                    switch (question.qtype) {
                                        case "choice":
                                        case "select":
                                            qObj.correct_answer = ArrayMerge(ArraySelect(question.variant, "This.correct == '1'"), "HtmlToPlainText(This.text)", "; ");
                                            qObj.answer = ArrayMerge(ArraySelect(question.variant, "Trim(This) == '1'"), "HtmlToPlainText(This.text)", "; ");
                                            break;
                                        default:
                                            qObj.answer = ArrayMerge(question.variant, "HtmlToPlainText(This.value)", "; ");
                                            qObj.correct_answer = ArrayMerge(question.variant, "HtmlToPlainText(This.cor_value)", "; ");
                                            break;
                                    }
                                    obj.questions[qid] = qObj;
                                }
                            }
                        }
                    }
                } catch(e) {
                    alert("Ошибка при разборе анналов: " + e.message);
                }
                
                aLearnings.push(obj);
            }
        }
    }
    
    // Генерация HTML (без изменений, как в предыдущем варианте)
    var html = new Binary();
    html.AppendStr("<html xmlns:o=\"urn:schemas-microsoft-com:office:office\" xmlns:x=\"urn:schemas-microsoft-com:office:excel\" xmlns=\"http://www.w3.org/TR/REC-html40\">" +
        "<head><meta http-equiv=\"Content-Type\" content=\"text/html; charset=utf-8\"/>" +
        "<meta name=\"ProgId\" content=\"Excel.Sheet\"/><meta name=\"Generator\" content=\"Microsoft Excel 11\"/>" +
        "<style>td.two-decimals {mso-number-format: \"0\\.00\"}</style>" +
        "</head><body><table border=\"1\" cellpadding=\"2\" cellspacing=\"0\">");
    
    if (ArrayCount(aLearnings) == 0) {
        html.AppendStr("<tr><td colspan='9'>Нет данных за выбранный период</td></tr>");
    } else {
        // Шапка с вопросами
        if (ArrayCount(aQuestions) > 0) {
            html.AppendStr("<tr><td colspan='9'></td>");
            for (var qi = 0; qi < ArrayCount(aQuestions); qi++) {
                html.AppendStr("<td colspan='4' bgcolor='#EA9999'><b>" + aQuestions[qi].text + "</b></td>");
            }
            html.AppendStr("</tr>");
        }
        
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
        
        for (var qi = 0; qi < ArrayCount(aQuestions); qi++) {
            html.AppendStr("<td bgcolor='#FCE5CD'><b>Тип</b></td>");
            html.AppendStr("<td bgcolor='#FCE5CD'><b>Результат</b></td>");
            html.AppendStr("<td bgcolor='#FCE5CD'><b>Полученный ответ</b></td>");
            html.AppendStr("<td bgcolor='#FCE5CD'><b>Правильный ответ</b></td>");
        }
        html.AppendStr("</tr>");
        
        for (var c = 0; c < ArrayCount(aLearnings); c++) {
            var K = aLearnings[c];
            html.AppendStr("<tr valign='top'>");
            html.AppendStr("<td width='250'>" + (K.person_fullname || "") + "</td>");
            html.AppendStr("<td width='200'>" + (K.assessment_name || "") + "</td>");
            html.AppendStr("<td width='100'>" + (K.person_code || "") + "</td>");
            html.AppendStr("<td width='150'>" + (K.org || "") + "</td>");
            html.AppendStr("<td width='150'>" + (K.subdivision || "") + "</td>");
            html.AppendStr("<td width='150'>" + (K.position || "") + "</td>");
            html.AppendStr("<td width='110' align='center'>" + StrDate(K.date, false, false) + "</td>");
            html.AppendStr("<td width='100' align='center'>" + (K.status || "") + "</td>");
            
            var score = "";
            if (K.max_score != null && K.max_score != 0) {
                var percent = Math.round(100 * K.score / K.max_score);
                score = K.score + " (" + percent + "%)";
            } else {
                score = K.score;
            }
            html.AppendStr("<td width='100' align='center'>" + score + "</td>");
            
            for (var qi = 0; qi < ArrayCount(aQuestions); qi++) {
                var qid = aQuestions[qi].id;
                var qData = K.questions[qid];
                if (qData == undefined) {
                    html.AppendStr("<td colspan='4'> </td>");
                } else {
                    var bg = (qData.result == "неверно") ? "#F4CCCC" : "#D9EAD3";
                    html.AppendStr("<td width='150'>" + (qData.quest_type || "") + "</td>");
                    html.AppendStr("<td width='80' align='center' bgcolor='" + bg + "'>" + (qData.result || "") + "</td>");
                    html.AppendStr("<td width='250'>" + (qData.answer || "") + "</td>");
                    html.AppendStr("<td width='250'>" + (qData.correct_answer || "") + "</td>");
                }
            }
            html.AppendStr("</tr>");
        }
    }
    
    html.AppendStr("</table></body></html>");
    alert("конец функции createReportString");
    return html.GetStr();
}
