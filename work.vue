// --- прочие требования ---

var d = Trim(getFormField("duties", ""));
var dh = Trim(getFormField("duties_history", ""));
var dutiesValue = "";

// 1. Если пользователь вручную ввёл duties — берём его
if (d != "") {
    dutiesValue = d;
}
// 2. Иначе, если передан duties_history — это ID заявки
else if (dh != "") {

    try {
        var otherDoc = tools.open_doc(OptInt(dh)).TopElem;
        dutiesValue = Trim(otherDoc.custom_elems.ObtainChildByKey("f_candidate_function").value);
    }
    catch (e) {
        dutiesValue = ""; // если документ не открылся — пусто
    }
}

// 3. Записываем итоговое значение в текущую заявку
oReqDocTE.custom_elems.ObtainChildByKey("f_candidate_function").value = dutiesValue;


// --- требования (requirements) по аналогии ---
var r = Trim(getFormField("requirements", ""));
var rh = Trim(getFormField("requirements_history", ""));
var reqValue = "";

if (r != "") {
    reqValue = r;
}
else if (rh != "") {
    try {
        var otherDoc2 = tools.open_doc(OptInt(rh)).TopElem;
        reqValue = Trim(otherDoc2.custom_elems.ObtainChildByKey("f_candidate_quality").value);
    }
    catch (e) {
        reqValue = "";
    }
}

oReqDocTE.custom_elems.ObtainChildByKey("f_candidate_quality").value = reqValue;
