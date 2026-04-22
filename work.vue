else
{
      objectsData = annals.au.history.objects;
      objectsArr = null;

    // Определяем, где лежит список объектов
    if (ArrayCount(objectsData) > 0) {
        objectsArr = objectsData;
    } else if (objectsData.object != undefined) {
        objectsArr = objectsData.object;
    } else {
        alert("Неизвестная структура objects");
    }

    if (objectsArr != null) {
        alert("objectsArr count = " + ArrayCount(objectsArr));
        // Перебираем объекты (for..in вернёт сами объекты)
        for (  objItem in objectsArr) {
            if (objItem.section == undefined) continue;

              sectionsData = objItem.section;
            // sectionsData может быть массивом или объектом
            if (ArrayCount(sectionsData) > 0) {
                // Перебираем секции
                for (  sec in sectionsData) {
                    if (sec.question == undefined) continue;
                      questionsData = sec.question;
                    // Перебираем вопросы
                    if (ArrayCount(questionsData) > 0) {
                        for (  q in questionsData) {
                            processQuestion(q);
                        }
                    } else if (questionsData.question != undefined) {
                        for (  q in questionsData.question) {
                            processQuestion(q);
                        }
                    } else {
                        processQuestion(questionsData);
                    }
                }
            } else if (sectionsData.section != undefined) {
                for (  sec in sectionsData.section) {
                    // аналогично
                    if (sec.question == undefined) continue;
                      questionsData = sec.question;
                    // ... обработать вопросы
                }
            } else {
                // одна секция
                  sec = sectionsData;
                if (sec.question != undefined) {
                      questionsData = sec.question;
                    if (ArrayCount(questionsData) > 0) {
                        for (  q in questionsData) {
                            processQuestion(q);
                        }
                    } else {
                        processQuestion(questionsData);
                    }
                }
            }
        }
    }

    // Вспомогательная функция для обработки одного вопроса
    function processQuestion(q) {
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
           iants = [];
        if (q. iant != undefined) {
            if (ArrayCount(q. iant) > 0) {
                for (  v in q. iant) {
                     iants.push(v);
                }
            } else {
                 iants.push(q. iant);
            }
        }

        // Правильный ответ (ws_right)
          correctTexts = [];
        for (  v in  iants) {
            if (v.ws_right == '1')
                correctTexts.push(HtmlToPlainText(v.text || ""));
        }
        qObj.correct_answer = correctTexts.join("; ");

        // Ответ пользователя (value)
          userTexts = [];
        for (  v in  iants) {
            if (v.value != undefined && v.value != "")
                userTexts.push(HtmlToPlainText(v.text || ""));
        }
        if (userTexts.length == 0) {
            for (  v in  iants) {
                if (v.selected == '1' || v.selected == true)
                    userTexts.push(HtmlToPlainText(v.text || ""));
            }
        }
        qObj.answer = userTexts.join("; ");

        obj.questions[qid] = qObj;
    }
}
