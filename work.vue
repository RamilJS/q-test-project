for (var q in questions) {
    var question = questions[q];
    var qid = String(question.ident || question.id);
    
    // Уникальный список вопросов
    if (ArrayOptFind(aQuestions, "This.id == '" + qid + "'") == undefined) {
        aQuestions.push({
            id: qid,
            text: HtmlToPlainText(question.text || "")
        });
    }
    
    var qObj = new Object();
    
    // --- Тип вопроса ---
    try {
        if (question.qtype != undefined && question.qtype.OptForeignElem != undefined)
            qObj.quest_type = question.qtype.ForeignElem.name;
        else
            qObj.quest_type = question.qtype || "";
    } catch(e) {
        qObj.quest_type = "";
        LogEvent("Ramil", "Ошибка типа вопроса: " + e.message);
    }
    
    // --- Результат (верно/неверно) ---
    try {
        qObj.result = tools_web.is_correct_question(question) ? "верно" : "неверно";
    } catch(e) {
        qObj.result = "";
        LogEvent("Ramil", "Ошибка результата: " + e.message);
    }
    
    // --- Нормализация вариантов ---
    var variants = [];
    if (question.variant != undefined) {
        if (ObjectType(question.variant) == "array")
            variants = question.variant;
        else
            variants = [question.variant];
    }
    
    // --- Правильный ответ (используем ws_right) ---
    var correctTexts = [];
    for (var vi = 0; vi < ArrayCount(variants); vi++) {
        var v = variants[vi];
        if (v.ws_right == '1')
            correctTexts.push(HtmlToPlainText(v.text || ""));
    }
    qObj.correct_answer = correctTexts.join("; ");
    
    // --- Ответ пользователя (используем value) ---
    var userTexts = [];
    for (var vi = 0; vi < ArrayCount(variants); vi++) {
        var v = variants[vi];
        if (v.value != undefined && v.value != "")
            userTexts.push(HtmlToPlainText(v.text || ""));
    }
    // Если по value не нашли, пробуем selected
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
