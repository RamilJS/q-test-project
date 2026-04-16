qObj = new Object();

// --- ТИП ВОПРОСА ---
if (q.qtype != undefined && q.qtype.OptForeignElem != undefined)
    qObj.quest_type = q.qtype.ForeignElem.name;
else
    qObj.quest_type = q.qtype;


// --- РЕЗУЛЬТАТ ---
try
{
    qObj.result = tools_web.is_correct_question(q) ? "верно" : "неверно";
}
catch(e)
{
    qObj.result = "";
}


// --- НОРМАЛИЗАЦИЯ VARIANT ---
var variants = [];

if (q.variant != undefined)
{
    if (ObjectType(q.variant) == "array")
        variants = q.variant;
    else
        variants = [q.variant];
}


// --- ПРАВИЛЬНЫЙ ОТВЕТ ---
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
        var _str = "";
        var _counter = 0;

        for (var ii = 0; ii < ArrayCount(variants); ii++)
        {
            if (_str != "")
                _str += "; ";

            _str += _counter;
            _counter++;
        }

        qObj.correct_answer = _str;
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


// --- ОТВЕТ ПОЛЬЗОВАТЕЛЯ ---
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
