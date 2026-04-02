var oSelect_collaborator_ids = getFormField('select_collaborator_ids', "");
if (oSelect_collaborator_ids != "") {
    oSelect_collaborator_ids = oSelect_collaborator_ids.split(";");
} else {
    oSelect_collaborator_ids = [];
}

var oSelect_assessment_ids = getFormField('select_assessment_ids', "");
if (oSelect_assessment_ids != "") {
    oSelect_assessment_ids = oSelect_assessment_ids.split(";");
} else {
    oSelect_assessment_ids = [];
}


// -------------------- УТИЛИТЫ --------------------

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


// -------------------- ГЛАВНАЯ ФУНКЦИЯ ОТЧЁТА --------------------

function createReportString(aCollaboratorIDArray, aAsessmentIDArray, dStartDate, dFinishDate){

    var reportStr = new Binary();
    reportStr.AppendStr("<html><body><table border='1' cellpadding='2' cellspacing='0'>");

    var aLearnings = [];
    var aQuestions = [];

    // --- СБОР ДАННЫХ ---
    for (person_id in aCollaboratorIDArray) {
        for (ass_id in aAsessmentIDArray) {

            var arr = XQuery(
                "for $l in test_learnings " +
                "where $l/person_id = " + person_id +
                " and $l/assessment_id = " + ass_id +
                " and $l/start_usage_date >= date('" + StrDate(dStartDate, false) + "')" +
                " and $l/start_usage_date <= date('" + StrDate(dFinishDate, false) + "')" +
                " return $l"
            );

            for (l in arr) {

                var learningDoc = OpenDoc(UrlFromDocID(l.id)).TopElem;

                var obj = new Object();
                obj.person_fullname = l.person_id.ForeignElem.fullname;
                obj.person_code = l.person_id.ForeignElem.code;
                obj.org = l.person_id.ForeignElem.org_name;
                obj.position = l.person_id.ForeignElem.position_name;
                obj.test_name = l.assessment_id.ForeignElem.name;
                obj.score = l.score;
                obj.max_score = l.max_score;
                obj.start_date = l.start_usage_date;

                obj.questions = {};

                var fldAnnals = tools.annals_decrypt(learningDoc);

                if (fldAnnals != null) {

                    var history = fldAnnals.au.history;

                    if (ArrayCount(history.objects) > 0) {

                        var sections = history.objects[0].section;

                        for (s in sections) {
                            for (q in s.question) {

                                // список вопросов
                                if (ArrayOptFind(aQuestions, "This.id == " + q.id) == undefined) {
                                    aQuestions.push({
                                        id: q.id,
                                        text: HtmlToPlainText(q.text)
                                    });
                                }

                                var curQ = new Object();

                                curQ.type = (q.qtype.OptForeignElem != undefined ? q.qtype.ForeignElem.name : "");
                                curQ.result = tools_web.is_correct_question(q) ? "Верно" : "Неверно";
                                curQ.correct = ArrayMerge(q.variant, "HtmlToPlainText(cor_value)", "; ");
                                curQ.answer = ArrayMerge(q.variant, "HtmlToPlainText(value)", "; ");

                                obj.questions[q.id] = curQ;
                            }
                        }
                    }
                }

                aLearnings.push(obj);
            }
        }
    }

    // --- HEADER 1 ---
    reportStr.AppendStr("<tr>");
    for (i = 0; i < 8; i++) reportStr.AppendStr("<td></td>");
    for (q in aQuestions) {
        reportStr.AppendStr("<td colspan='4'><b>" + q.text + "</b></td>");
    }
    reportStr.AppendStr("</tr>");

    // --- HEADER 2 ---
    reportStr.AppendStr("<tr>");

    reportStr.AppendStr("<td><b>ФИО</b></td>");
    reportStr.AppendStr("<td><b>Код</b></td>");
    reportStr.AppendStr("<td><b>Орг</b></td>");
    reportStr.AppendStr("<td><b>Должность</b></td>");
    reportStr.AppendStr("<td><b>Тест</b></td>");
    reportStr.AppendStr("<td><b>Дата</b></td>");
    reportStr.AppendStr("<td><b>Баллы</b></td>");
    reportStr.AppendStr("<td><b>%</b></td>");

    for (q in aQuestions) {
        reportStr.AppendStr("<td><b>Тип</b></td>");
        reportStr.AppendStr("<td><b>Результат</b></td>");
        reportStr.AppendStr("<td><b>Ответ</b></td>");
        reportStr.AppendStr("<td><b>Правильный</b></td>");
    }

    reportStr.AppendStr("</tr>");

    // --- ДАННЫЕ ---
    for (l in aLearnings) {

        reportStr.AppendStr("<tr>");

        var percent = (l.max_score > 0) ? (l.score / l.max_score * 100) : 0;

        reportStr.AppendStr("<td>" + l.person_fullname + "</td>");
        reportStr.AppendStr("<td>" + l.person_code + "</td>");
        reportStr.AppendStr("<td>" + l.org + "</td>");
        reportStr.AppendStr("<td>" + l.position + "</td>");
        reportStr.AppendStr("<td>" + l.test_name + "</td>");
        reportStr.AppendStr("<td>" + StrDate(l.start_date, true, false) + "</td>");
        reportStr.AppendStr("<td>" + l.score + " / " + l.max_score + "</td>");
        reportStr.AppendStr("<td>" + StrReal(percent, 1) + "%</td>");

        for (q in aQuestions) {

            var curQ = l.questions[q.id];

            if (curQ == undefined) {
                reportStr.AppendStr("<td colspan='4'></td>");
            } else {

                var color = (curQ.result == "Неверно") ? "#ffe6e6" : "#e6ffe6";

                reportStr.AppendStr("<td>" + curQ.type + "</td>");
                reportStr.AppendStr("<td style='background:" + color + "'>" + curQ.result + "</td>");
                reportStr.AppendStr("<td>" + curQ.answer + "</td>");
                reportStr.AppendStr("<td>" + curQ.correct + "</td>");
            }
        }

        reportStr.AppendStr("</tr>");
    }

    reportStr.AppendStr("</table></body></html>");

    return reportStr.GetStr();
}



2 вариант
function createReportString(aCollaboratorIDArray, aAsessmentIDArray, dStartDate, dFinishDate){

    var reportStr = new Binary();

    var Ps = new Object();
    Ps.questions = [];
    Ps.learnings = [];

    // ---------------- СБОР ДАННЫХ (АНАЛОГ CMD=RUN) ----------------

    for (person_id in aCollaboratorIDArray) {
        for (ass_id in aAsessmentIDArray) {

            var arr = XQuery(
                "for $l in test_learnings " +
                "where $l/person_id = " + person_id +
                " and $l/assessment_id = " + ass_id +
                " and $l/start_usage_date >= date('" + StrDate(dStartDate, false) + "')" +
                " and $l/start_usage_date <= date('" + StrDate(dFinishDate, false) + "')" +
                " return $l"
            );

            for (l in arr) {

                var learningDoc = OpenDoc(UrlFromDocID(l.id)).TopElem;

                var obj = new Object();

                obj.person_fullname = l.person_id.ForeignElem.fullname;
                obj.person_code = l.person_id.ForeignElem.code;
                obj.person_position_name = l.person_id.ForeignElem.position_name;
                obj.person_subdivision_name = l.person_id.ForeignElem.subdivision_name;
                obj.person_id = l.person_id;
                obj.start_usage_date = l.start_usage_date;
                obj.state_id = l.state_id;
                obj.score = l.score;
                obj.max_score = l.max_score;

                obj.questions = {};

                var fldAnnals = tools.annals_decrypt(learningDoc);

                if (fldAnnals != null) {

                    var history = fldAnnals.au.history;

                    if (ArrayCount(history.objects) > 0) {

                        var sections = history.objects[0].section;

                        for (s in sections) {
                            for (q in s.question) {

                                // --- список вопросов
                                if (ArrayOptFind(Ps.questions, "This.id == " + q.id) == undefined) {
                                    Ps.questions.push({
                                        id: q.id,
                                        text: HtmlToPlainText(q.text)
                                    });
                                }

                                var curQ = new Object();

                                curQ.quest_type = (q.qtype.OptForeignElem != undefined ? q.qtype.ForeignElem.name : "");
                                curQ.result = tools_web.is_correct_question(q) ? "Верно" : "Неверно";
                                curQ.correct_answer = ArrayMerge(q.variant, "HtmlToPlainText(cor_value)", "; ");
                                curQ.answer = ArrayMerge(q.variant, "HtmlToPlainText(value)", "; ");

                                obj.questions[q.id] = curQ;
                            }
                        }
                    }
                }

                Ps.learnings.push(obj);
            }
        }
    }

    // ---------------- HTML (1 В 1 ИЗ ШАБЛОНА) ----------------

    reportStr.AppendStr("<html><body><table border='1' cellpadding='2' cellspacing='0'>");

    // HEADER 1
    reportStr.AppendStr("<tr>");
    reportStr.AppendStr("<td colspan='8'></td>");

    for (q in Ps.questions) {
        reportStr.AppendStr("<td colspan='4' bgcolor='#FF4433'><b>" + q.text + "</b></td>");
    }

    reportStr.AppendStr("</tr>");

    // HEADER 2
    reportStr.AppendStr("<tr>");

    reportStr.AppendStr("<td><b>ФИО</b></td>");
    reportStr.AppendStr("<td><b>Код</b></td>");
    reportStr.AppendStr("<td><b>Орг</b></td>");
    reportStr.AppendStr("<td><b>Подразделение</b></td>");
    reportStr.AppendStr("<td><b>Должность</b></td>");
    reportStr.AppendStr("<td><b>Дата</b></td>");
    reportStr.AppendStr("<td><b>Статус</b></td>");
    reportStr.AppendStr("<td><b>Баллы</b></td>");

    for (q in Ps.questions) {
        reportStr.AppendStr("<td><b>Тип</b></td>");
        reportStr.AppendStr("<td><b>Результат</b></td>");
        reportStr.AppendStr("<td><b>Ответ</b></td>");
        reportStr.AppendStr("<td><b>Правильный</b></td>");
    }

    reportStr.AppendStr("</tr>");

    // DATA
    for (l in Ps.learnings) {

        reportStr.AppendStr("<tr>");

        var percent = (l.max_score > 0) ? (l.score / l.max_score * 100) : 0;

        reportStr.AppendStr("<td>" + l.person_fullname + "</td>");
        reportStr.AppendStr("<td>" + l.person_code + "</td>");
        reportStr.AppendStr("<td>" + l.person_id.ForeignElem.org_name + "</td>");
        reportStr.AppendStr("<td>" + l.person_subdivision_name + "</td>");
        reportStr.AppendStr("<td>" + l.person_position_name + "</td>");
        reportStr.AppendStr("<td>" + StrDate(l.start_usage_date, true, false) + "</td>");
        reportStr.AppendStr("<td>" + l.state_id.ForeignElem.name + "</td>");
        reportStr.AppendStr("<td>" + l.score + " (" + StrReal(percent,1) + "%)</td>");

        for (q in Ps.questions) {

            var curQ = l.questions[q.id];

            if (curQ == undefined) {
                reportStr.AppendStr("<td></td><td></td><td></td><td></td>");
            } else {

                var color = (curQ.result == "Неверно") ? "#FFCCCC" : "#CCFFCC";

                reportStr.AppendStr("<td>" + curQ.quest_type + "</td>");
                reportStr.AppendStr("<td bgcolor='" + color + "'>" + curQ.result + "</td>");
                reportStr.AppendStr("<td>" + curQ.answer + "</td>");
                reportStr.AppendStr("<td>" + curQ.correct_answer + "</td>");
            }
        }

        reportStr.AppendStr("</tr>");
    }

    reportStr.AppendStr("</table></body></html>");

    return reportStr.GetStr();
}

