if (annals == null) {
    alert("annals == null");
} else if (annals.au == undefined) {
    alert("annals.au undefined");
} else if (annals.au.history == undefined) {
    alert("history undefined");
} else if (annals.au.history.objects == undefined) {
    alert("objects undefined");
} else {
    var objectsData = annals.au.history.objects;
    var objectsList = [];

    // Если objectsData — массив
    if (ArrayCount(objectsData) > 0) {
        objectsList = objectsData;
    }
    // Если objectsData — объект с полем object (и это массив)
    else if (objectsData.object != undefined && ArrayCount(objectsData.object) > 0) {
        objectsList = objectsData.object;
    }
    // Если objectsData — одиночный объект (не массив и нет поля object)
    else if (objectsData.section != undefined) {
        objectsList = [objectsData];
    }

    alert("objectsList count = " + ArrayCount(objectsList));

    for (var oi = 0; oi < ArrayCount(objectsList); oi++) {
        var objItem = objectsList[oi];
        if (objItem.section == undefined) continue;

        var sectionsData = objItem.section;
        var sectionsList = [];
        if (ArrayCount(sectionsData) > 0) {
            sectionsList = sectionsData;
        } else if (sectionsData.section != undefined) {
            sectionsList = sectionsData.section;
        } else {
            sectionsList = [sectionsData];
        }

        alert("sections count = " + ArrayCount(sectionsList));

        for (var si = 0; si < ArrayCount(sectionsList); si++) {
            var sec = sectionsList[si];
            if (sec.question == undefined) continue;

            var questionsData = sec.question;
            var questionsList = [];
            if (ArrayCount(questionsData) > 0) {
                questionsList = questionsData;
            } else if (questionsData.question != undefined) {
                questionsList = questionsData.question;
            } else {
                questionsList = [questionsData];
            }

            for (var qi = 0; qi < ArrayCount(questionsList); qi++) {
                var q = questionsList[qi];
                alert("question найден: " + (q.ident || q.id));

                var qid = String(q.ident || q.id);

                if (ArrayOptFind(aQuestions, "This.id == '" + qid + "'") == undefined) {
                    aQuestions.push({
                        id: qid,
                        text: HtmlToPlainText(q.text || "")
                    });
                }

                var qObj = new Object();

                // Тип вопроса
                try {
                    if (q.qtype != undefined && q.qtype.OptForeignElem != undefined)
                        qObj.quest_type = q.qtype.ForeignElem.name;
                    else
                        qObj.quest_type = q.qtype || "";
                } catch(e) {
                    qObj.quest_type = "";
                }

                // Результат
                try {
                    qObj.result = tools_web.is_correct_question(q) ? "верно" : "неверно";
                } catch(e) {
                    qObj.result = "";
                }

                // Нормализация вариантов
                var variants = [];
                if (q.variant != undefined) {
                    if (ObjectType(q.variant) == "array")
                        variants = q.variant;
                    else
                        variants = [q.variant];
                }

                // Правильный ответ (ws_right)
                var correctTexts = [];
                for (var vi = 0; vi < ArrayCount(variants); vi++) {
                    var v = variants[vi];
                    if (v.ws_right == '1')
                        correctTexts.push(HtmlToPlainText(v.text || ""));
                }
                qObj.correct_answer = correctTexts.join("; ");

                // Ответ пользователя (value)
                var userTexts = [];
                for (var vi = 0; vi < ArrayCount(variants); vi++) {
                    var v = variants[vi];
                    if (v.value != undefined && v.value != "")
                        userTexts.push(HtmlToPlainText(v.text || ""));
                }
                if (userTexts.length == 0) {
                    for (var vi = 0; vi < ArrayCount(variants); vi++) {
                        var v = variants[vi];
                        if (v.selected == '1' || v.selected == true)
                            userTexts.push(HtmlToPlainText(v.text || ""));
                    }
                }
                qObj.answer = userTexts.join("; ");

                obj.questions[qid] = qObj;
            }
        }
    }
}
