// --- вопросы ---
obj.questions = [];

for (var oi = 0; oi < ArrayCount(objectsArr); oi++)
{
    var objItem = objectsArr[oi];

    if (objItem.section == undefined)
        continue;

    var sections = objItem.section;

    if (ObjectType(sections) != "array")
        sections = [sections];

    for (var si = 0; si < ArrayCount(sections); si++)
    {
        var sec = sections[si];

        if (sec.question == undefined)
            continue;

        var questions = sec.question;

        if (ObjectType(questions) != "array")
            questions = [questions];

        for (var qi = 0; qi < ArrayCount(questions); qi++)
        {
            var q = questions[qi];
            var qid = String(q.ident);

            // --- список всех вопросов ---
            if (ArrayOptFind(aQuestions, "This.id == '" + qid + "'") == undefined)
            {
                aQuestions.push({
                    id: qid,
                    text: HtmlToPlainText(q.text)
                });
            }

            var qObj = new Object();
            qObj.id = qid;

            // --- тип ---
            qObj.quest_type = (q.qtype.OptForeignElem != undefined)
                ? q.qtype.ForeignElem.name
                : q.qtype;

            // --- результат ---
            try
            {
                qObj.result = tools_web.is_correct_question(q) ? "верно" : "неверно";
            }
            catch(e)
            {
                qObj.result = "";
            }

            // --- нормализация вариантов ---
            var variants = [];

            if (q.variant != undefined)
            {
                if (ObjectType(q.variant) == "array")
                    variants = q.variant;
                else
                    variants = [q.variant];
            }

            // --- правильный ответ ---
            switch (q.qtype)
            {
                case "choice":
                case "select":
                    qObj.correct_answer = ArrayMerge(
                        ArraySelect(variants, "correct == '1'"),
                        "HtmlToPlainText(text)",
                        "; "
                    );
                    break;

                case "range":
                    var tmp = "";
                    for (var ii = 0; ii < ArrayCount(variants); ii++)
                        tmp += (tmp == "" ? "" : "; ") + ii;
                    qObj.correct_answer = tmp;
                    break;

                case "numeric":
                    qObj.correct_answer = ArrayMerge(
                        variants,
                        "ArrayMerge(cond,'operator.ForeignElem.name + Trim(Value)',', ')",
                        "; "
                    );
                    break;

                case "text":
                    qObj.correct_answer = ArrayMerge(
                        variants,
                        "ArrayMerge(cond,'operator==\\'cn\\'?\\'...\\'+Trim(Value)+\\'...\\':Trim(Value)',', ')",
                        "; "
                    );
                    break;

                default:
                    qObj.correct_answer = ArrayMerge(
                        variants,
                        "HtmlToPlainText(cor_value)",
                        "; "
                    );
                    break;
            }

            // --- ответ пользователя ---
            switch (q.qtype)
            {
                case "choice":
                case "select":
                    qObj.answer = ArrayMerge(
                        ArraySelect(variants, "Trim(This) == '1'"),
                        "HtmlToPlainText(text)",
                        "; "
                    );
                    break;

                case "range":
                    qObj.answer = ArrayMerge(
                        ArraySelect(variants, "Trim(value) != ''"),
                        "HtmlToPlainText(value)",
                        "; "
                    );
                    break;

                default:
                    qObj.answer = ArrayMerge(
                        variants,
                        "HtmlToPlainText(value)",
                        "; "
                    );
                    break;
            }

            // --- добавляем (НЕ object, а массив!) ---
            obj.questions.push(qObj);
        }
    }
}
