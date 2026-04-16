// После получения annals
var fldAnnals = annals; // tools.annals_decrypt(doc)

if (fldAnnals == null) continue; // или обработать

var _cur_history = fldAnnals.au.history;
if (ArrayCount(_cur_history.objects) == 0) continue;
var _cur_sections = ArrayFirstElem(_cur_history.objects).section;

for (var _section in _cur_sections) {
    var _sectionObj = _cur_sections[_section];
    if (!_sectionObj.question) continue;
    for (var _question in _sectionObj.question) {
        var q = _sectionObj.question[_question];
        var qid = String(q.PrimaryKey || q.ident || q.id);
        
        // Добавление в список вопросов (уникально)
        if (ArrayOptFind(aQuestions, "This.id == '" + qid + "'") == undefined) {
            aQuestions.push({
                id: qid,
                text: HtmlToPlainText(q.text || "")
            });
        }
        
        var qObj = new Object();
        
        // Тип вопроса
        var bIsWeight = (ArrayOptFind(q.variant, 'varscore.HasValue') != undefined);
        qObj.quest_type = (q.qtype && q.qtype.OptForeignElem ? q.qtype.ForeignElem.name : (q.qtype || "")) + (bIsWeight ? " взвешенный" : "");
        
        // Результат
        qObj.result = (!bIsWeight ? (tools_web.is_correct_question(q) ? "верно" : "неверно") : "");
        
        // --- Правильный ответ (correct_answer) ---
        switch (q.qtype) {
            case "choice":
            case "select":
                qObj.correct_answer = ArrayMerge(
                    ArraySelect(q.variant, "correct == '1'"),
                    "HtmlToPlainText(text)",
                    "; "
                );
                break;
            case "range":
                var _str = "";
                var _counter = 0;
                for (var _ans in q.variant) {
                    _str += (_str == "" ? "" : "; ") + _counter++;
                }
                qObj.correct_answer = _str;
                break;
            case "numeric":
                qObj.correct_answer = ArrayMerge(
                    q.variant,
                    "ArrayMerge(cond,'operator.ForeignElem.name + Trim(Value)',', ')",
                    "; "
                );
                break;
            case "text":
                qObj.correct_answer = ArrayMerge(
                    q.variant,
                    "ArrayMerge(cond,'operator=='cn'?'...'+Trim(Value)+'...':Trim(Value)',', ')",
                    "; "
                );
                break;
            default:
                qObj.correct_answer = ArrayMerge(
                    q.variant,
                    "HtmlToPlainText(cor_value)",
                    "; "
                );
                break;
        }
        
        // --- Ответ пользователя (answer) ---
        switch (q.qtype) {
            case "choice":
            case "select":
                qObj.answer = ArrayMerge(
                    ArraySelect(q.variant, "Trim(This) == '1'"),
                    "HtmlToPlainText(text)",
                    "; "
                );
                break;
            case "range":
                qObj.answer = ArrayMerge(
                    ArraySelect(q.variant, "Trim(value) != ''"),
                    "HtmlToPlainText(value)",
                    "; "
                );
                break;
            default:
                qObj.answer = ArrayMerge(
                    q.variant,
                    "HtmlToPlainText(value)",
                    "; "
                );
                break;
        }
        
        obj.questions[qid] = qObj;
    }
}
