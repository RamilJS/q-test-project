for (var k = 0; k < ArrayCount(arr); k++) {
    var learning = arr[k];
    alert("Обработка learning.id = " + learning.id);

    var obj = new Object();
    // Поля из твоего XQuery (как в логах)
    obj.person_fullname = learning.person_fullname;
    obj.assessment_name = learning.assessment_name;
    obj.person_code = (learning.person_id && learning.person_id.ForeignElem) ? learning.person_id.ForeignElem.code : "";
    obj.org = learning.person_org_name;
    obj.subdivision = learning.person_subdivision_name;
    obj.position = learning.person_position_name;
    obj.date = learning.start_usage_date;
    obj.status = (learning.state_id && learning.state_id.ForeignElem) ? learning.state_id.ForeignElem.name : learning.state_id;
    obj.score = learning.score;
    obj.max_score = learning.max_score;
    obj.questions = {};

    // --- Расшифровка анналов (как в примере) ---
    try {
        alert("Открываем документ l.id = " + learning.id);
        var doc = OpenDoc(UrlFromDocID(learning.id)).TopElem;
        alert("doc открыт");

        var annals = tools.annals_decrypt(doc);
        alert("annals = " + (annals ? "не пусто" : "пусто"));

        if (annals == null) {
            alert("annals == null, пропускаем");
            continue;
        }

        var history = annals.au.history;
        if (!history || ArrayCount(history.objects) == 0) {
            alert("Нет history.objects");
            continue;
        }

        // Берём первый объект истории (как в примере)
        var sections = ArrayFirstElem(history.objects).section;
        if (!sections) {
            alert("Нет sections");
            continue;
        }

        // Перебираем секции (используем индексный цикл)
        for (var s = 0; s < ArrayCount(sections); s++) {
            var section = sections[s];
            var questions = section.question;
            if (!questions) continue;

            for (var q = 0; q < ArrayCount(questions); q++) {
                var question = questions[q];
                var qid = question.PrimaryKey ? String(question.PrimaryKey) : String(question.id);

                // Добавляем вопрос в общий список (уникальный)
                if (ArrayOptFind(aQuestions, "This.id == '" + qid + "'") == undefined) {
                    aQuestions.push({
                        id: qid,
                        text: HtmlToPlainText(question.text || "")
                    });
                }

                var qObj = new Object();
                // Тип вопроса
                qObj.quest_type = (question.qtype && question.qtype.ForeignElem) ? question.qtype.ForeignElem.name : (question.qtype || "");

                // Проверяем, есть ли веса у вариантов (как в примере)
                var hasWeight = (ArrayOptFind(question.variant, "This.varscore.HasValue") != undefined);
                if (!hasWeight) {
                    qObj.result = tools_web.is_correct_question(question) ? "верно" : "неверно";
                } else {
                    qObj.result = ""; // для взвешенных вопросов результат не "верно/неверно"
                }

                // Полученный ответ (по аналогии с примером)
                var answer = "";
                var correct_answer = "";

                switch (question.qtype) {
                    case "choice":
                    case "select":
                        correct_answer = ArrayMerge(ArraySelect(question.variant, "This.correct == '1'"), "HtmlToPlainText(This.text)", "; ");
                        answer = ArrayMerge(ArraySelect(question.variant, "Trim(This) == '1'"), "HtmlToPlainText(This.text)", "; ");
                        break;
                    case "range":
                        var userVals = [];
                        for (var v = 0; v < ArrayCount(question.variant); v++) {
                            if (question.variant[v].value != undefined && question.variant[v].value != "")
                                userVals.push(HtmlToPlainText(question.variant[v].value));
                        }
                        answer = userVals.join("; ");
                        correct_answer = ""; // для range можно оставить пустым
                        break;
                    default:
                        answer = ArrayMerge(question.variant, "HtmlToPlainText(This.value)", "; ");
                        correct_answer = ArrayMerge(question.variant, "HtmlToPlainText(This.cor_value)", "; ");
                        break;
                }

                qObj.answer = answer;
                qObj.correct_answer = correct_answer;

                obj.questions[qid] = qObj;
            }
        }
    } catch(e) {
        alert("ERROR annals block: " + e.message);
    }

    aLearnings.push(obj);
}
