// --- Расшифровка анналов (как в рабочем примере) ---
try {
    var doc = OpenDoc(UrlFromDocID(learning.id)).TopElem;
    var annals = tools.annals_decrypt(doc);
    
    if (annals != null) {
        var history = annals.au.history;
        if (history && ArrayCount(history.objects) > 0) {
            var sections = ArrayFirstElem(history.objects).section;
            if (sections) {
                for (var s in sections) {
                    var section = sections[s];
                    var questions = section.question;
                    if (!questions) continue;
                    for (var q in questions) {
                        var question = questions[q];
                        var qid = question.PrimaryKey ? String(question.PrimaryKey) : String(question.id);
                        
                        if (ArrayOptFind(aQuestions, "This.id == '" + qid + "'") == undefined) {
                            aQuestions.push({
                                id: qid,
                                text: HtmlToPlainText(question.text || "")
                            });
                        }
                        
                        var qObj = new Object();
                        qObj.quest_type = (question.qtype && question.qtype.ForeignElem) ? question.qtype.ForeignElem.name : (question.qtype || "");
                        
                        var hasWeight = (ArrayOptFind(question.variant, "This.varscore.HasValue") != undefined);
                        if (!hasWeight) {
                            qObj.result = tools_web.is_correct_question(question) ? "верно" : "неверно";
                        } else {
                            qObj.result = "";
                        }
                        
                        // Формируем ответы (как в примере)
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
    }
} catch(e) {
    alert("Ошибка анналов: " + e.message);
}
