
try {
    alert("Открываем документ l.id = " + l.id);
    var doc = tools.open_doc(l.id).TopElem;
    alert("doc открыт");

    var annals = tools.annals_decrypt(doc);

    if (annals == null) {
        alert("annals == null");
    } else if (annals.au == undefined) {
        alert("annals.au undefined");
    } else if (annals.au.history == undefined) {
        alert("history undefined");
    } else if (annals.au.history.objects == undefined) {
        alert("objects undefined");
    } else {
        var objectsArr = annals.au.history.objects;
        // objectsArr может быть массивом или объектом с полем object
        var objList = [];
        if (ArrayCount(objectsArr) > 0) {
            objList = objectsArr;
        } else if (objectsArr.object != undefined) {
            objList = objectsArr.object;
            if (objList && !ArrayCount(objList)) objList = [objList];
        }

        for (var oi = 0; oi < ArrayCount(objList); oi++) {
            var objItem = objList[oi];
            if (objItem.section == undefined) continue;

            var sections = objItem.section;
            for (var si = 0; si < ArrayCount(sections); si++) {
                var sec = sections[si];
                if (sec.question == undefined) continue;

                var questions = sec.question;
                for (var qi = 0; qi < ArrayCount(questions); qi++) {
                    var q = questions[qi];
                    var qid = String(q.ident || q.id);

                    // Добавляем вопрос в общий список (уникальный)
                    if (ArrayOptFind(aQuestions, "This.id == '" + qid + "'") == undefined) {
                        aQuestions.push({
                            id: qid,
                            text: HtmlToPlainText(q.text || "")
                        });
                    }

                    var qObj = new Object();

                    // --- Тип вопроса ---
                    if (q.qtype != undefined && q.qtype.OptForeignElem != undefined)
                        qObj.quest_type = q.qtype.ForeignElem.name;
                    else
                        qObj.quest_type = q.qtype || "";

                    // --- Результат (верно/неверно) ---
                    try {
                        qObj.result = tools_web.is_correct_question(q) ? "верно" : "неверно";
                    } catch(e) {
                        qObj.result = "";
                    }

                    // --- Нормализация вариантов ответа ---
                    var variants = [];
                    if (q.variant != undefined) {
                        if (ObjectType(q.variant) == "array")
                            variants = q.variant;
                        else
                            variants = [q.variant];
                    }

                    // --- Правильный ответ (варианты с ws_right == '1') ---
                    var correctTexts = [];
                    for (var vi = 0; vi < ArrayCount(variants); vi++) {
                        var v = variants[vi];
                        if (v.ws_right == '1')
                            correctTexts.push(HtmlToPlainText(v.text || ""));
                    }
                    qObj.correct_answer = correctTexts.join("; ");

                    // --- Ответ пользователя (варианты, у которых поле value не пустое) ---
                    var userTexts = [];
                    for (var vi = 0; vi < ArrayCount(variants); vi++) {
                        var v = variants[vi];
                        // В зависимости от типа вопроса, выбранный ответ может быть в поле value
                        if (v.value != undefined && v.value != "")
                            userTexts.push(HtmlToPlainText(v.text || ""));
                    }
                    // Если ничего не нашли, возможно ответ хранится в поле selected
                    if (userTexts.length == 0) {
                        for (var vi = 0; vi < ArrayCount(variants); vi++) {
                            var v = variants[vi];
                            if (v.selected == '1' || v.selected == true)
                                userTexts.push(HtmlToPlainText(v.text || ""));
                        }
                    }
                    qObj.answer = userTexts.join("; ");

                    // Сохраняем данные ответа для этого прохождения
                    obj.questions[qid] = qObj;
                }
            }
        }
    }
} catch(e) {
    alert("ERROR annals block: " + e.message);
}
