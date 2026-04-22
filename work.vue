else
{
      objectsData = annals.au.history.objects;
      objectsArr = [];

    // Определяем массив объектов
    if (ArrayCount(objectsData) > 0) {
        for (  oi = 0; oi < ArrayCount(objectsData); oi++) {
            objectsArr.push(objectsData[oi]);
        }
    } else if (objectsData.object != undefined && ArrayCount(objectsData.object) > 0) {
        for (  oi = 0; oi < ArrayCount(objectsData.object); oi++) {
            objectsArr.push(objectsData.object[oi]);
        }
    } else if (objectsData.section != undefined) {
        objectsArr.push(objectsData);
    } else {
        alert("Неизвестная структура objects");
    }

    alert("objectsArr count = " + ArrayCount(objectsArr));

    for (  oi = 0; oi < ArrayCount(objectsArr); oi++) {
          objItem = objectsArr[oi];
        if (objItem.section == undefined) continue;

          sectionsData = objItem.section;
          sectionsArr = [];

        if (ArrayCount(sectionsData) > 0) {
            for (  si = 0; si < ArrayCount(sectionsData); si++) {
                sectionsArr.push(sectionsData[si]);
            }
        } else if (sectionsData.section != undefined) {
            for (  si = 0; si < ArrayCount(sectionsData.section); si++) {
                sectionsArr.push(sectionsData.section[si]);
            }
        } else {
            sectionsArr.push(sectionsData);
        }

        alert("sections count = " + ArrayCount(sectionsArr));

        for (  si = 0; si < ArrayCount(sectionsArr); si++) {
              sec = sectionsArr[si];
            if (sec.question == undefined) continue;

              questionsData = sec.question;
              questionsArr = [];

            if (ArrayCount(questionsData) > 0) {
                for (  qi = 0; qi < ArrayCount(questionsData); qi++) {
                    questionsArr.push(questionsData[qi]);
                }
            } else if (questionsData.question != undefined) {
                for (  qi = 0; qi < ArrayCount(questionsData.question); qi++) {
                    questionsArr.push(questionsData.question[qi]);
                }
            } else {
                questionsArr.push(questionsData);
            }

            for (  qi = 0; qi < ArrayCount(questionsArr); qi++) {
                  q = questionsArr[qi];
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
                        for (  vi = 0; vi < ArrayCount(q. iant); vi++) {
                             iants.push(q. iant[vi]);
                        }
                    } else {
                         iants.push(q. iant);
                    }
                }

                // Правильный ответ (ws_right)
                  correctTexts = [];
                for (  vi = 0; vi < ArrayCount( iants); vi++) {
                      v =  iants[vi];
                    if (v.ws_right == '1')
                        correctTexts.push(HtmlToPlainText(v.text || ""));
                }
                qObj.correct_answer = correctTexts.join("; ");

                // Ответ пользователя (value)
                  userTexts = [];
                for (  vi = 0; vi < ArrayCount( iants); vi++) {
                      v =  iants[vi];
                    if (v.value != undefined && v.value != "")
                        userTexts.push(HtmlToPlainText(v.text || ""));
                }
                if (userTexts.length == 0) {
                    for (  vi = 0; vi < ArrayCount( iants); vi++) {
                          v =  iants[vi];
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
