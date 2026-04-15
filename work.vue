
// --- Расшифровка анналов ---
try {
    alert("Открываем документ l.id = " + learning.id);
    var doc = OpenDoc(UrlFromDocID(learning.id)).TopElem;
    alert("doc открыт");
    
    var annals = tools.annals_decrypt(doc);
    alert("annals = " + (annals ? "не пусто" : "пусто"));
    
    // Проверяем наличие нужной структуры (без typeof)
    if (annals && annals.au && annals.au.history && annals.au.history.objects) {
        var objects = annals.au.history.objects;
        for (var objIdx = 0; objIdx < ArrayCount(objects); objIdx++) {
            var sections = objects[objIdx].section;
            if (!sections) continue;
            
            for (var s = 0; s < ArrayCount(sections); s++) {
                var section = sections[s];
                var questions = section.question;
                if (!questions) continue;
                
                for (var q = 0; q < ArrayCount(questions); q++) {
                    var question = questions[q];
                    var qid = String(question.id);
                    
                    if (ArrayOptFind(aQuestions, "This.id == '" + qid + "'") == undefined) {
                        aQuestions.push({
                            id: qid,
                            text: HtmlToPlainText(question.text || "")
                        });
                    }
                    
                    var qObj = new Object();
                    qObj.quest_type = (question.qtype && question.qtype.ForeignElem) ? question.qtype.ForeignElem.name : "";
                    qObj.result = tools_web.is_correct_question(question) ? "верно" : "неверно";
                    
                    var userAnswers = [];
                    var correctAnswers = [];
                    if (question.variant && ArrayCount(question.variant) > 0) {
                        for (var v = 0; v < ArrayCount(question.variant); v++) {
                            var variant = question.variant[v];
                            if (variant.value) userAnswers.push(HtmlToPlainText(variant.value));
                            if (variant.cor_value) correctAnswers.push(HtmlToPlainText(variant.cor_value));
                        }
                    }
                    qObj.answer = userAnswers.join("; ");
                    qObj.correct_answer = correctAnswers.join("; ");
                    
                    obj.questions[qid] = qObj;
                }
            }
        }
    } else {
        alert("Нет данных анналов или структура не соответствует ожидаемой");
    }
} catch(e) {
    alert("ERROR annals block: " + e.message);
}
