else
{
     objectsData = annals.au.history.objects;
     objectsList = [];

    // Приводим objectsData к массиву объектов
    if (ArrayCount(objectsData) > 0) {
        // objectsData уже массив — переложим в список
        for (objItem in objectsData) {
            objectsList.push(objItem);
        }
    } else if (objectsData.object != undefined && ArrayCount(objectsData.object) > 0) {
        for (objItem in objectsData.object) {
            objectsList.push(objItem);
        }
    } else if (objectsData.section != undefined) {
        objectsList.push(objectsData);
    }

    alert("objectsList count = " + ArrayCount(objectsList));

    for (objItem in objectsList) {
        if (objItem.section == undefined) continue;

         sectionsData = objItem.section;
         sectionsList = [];

        if (ArrayCount(sectionsData) > 0) {
            for (sec in sectionsData) {
                sectionsList.push(sec);
            }
        } else if (sectionsData.section != undefined) {
            for (sec in sectionsData.section) {
                sectionsList.push(sec);
            }
        } else {
            sectionsList.push(sectionsData);
        }

        alert("sections count = " + ArrayCount(sectionsList));

        for ( sec in sectionsList) {
            if (sec.question == undefined) continue;

            questionsData = sec.question;
            questionsList = [];

            if (ArrayCount(questionsData) > 0) {
                for ( q in questionsData) {
                    questionsList.push(q);
                }
            } else if (questionsData.question != undefined) {
                for ( q in questionsData.question) {
                    questionsList.push(q);
                }
            } else {
                questionsList.push(questionsData);
            }

            for ( q in questionsList) {
                alert("question найден: " + (q.ident || q.id));

                 qid = String(q.ident || q.id);

                if (ArrayOptFind(aQuestions, "This.id == '" + qid + "'") == undefined) {
                    aQuestions.push({
                        id: qid,
                        text: HtmlToPlainText(q.text || "")
                    });
                }

                 qObj = new Object();

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
                 variants = [];
                if (q.variant != undefined) {
                    if (ArrayCount(q.variant) > 0) {
                        for ( v in q.variant) {
                            variants.push(v);
                        }
                    } else {
                        variants.push(q.variant);
                    }
                }

                // Правильный ответ (ws_right)
                 correctTexts = [];
                for ( v in variants) {
                    if (v.ws_right == '1')
                        correctTexts.push(HtmlToPlainText(v.text || ""));
                }
                qObj.correct_answer = correctTexts.join("; ");

                // Ответ пользователя (value)
                 userTexts = [];
                for ( v in variants) {
                    if (v.value != undefined && v.value != "")
                        userTexts.push(HtmlToPlainText(v.text || ""));
                }
                if (userTexts.length == 0) {
                    for ( v in variants) {
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
