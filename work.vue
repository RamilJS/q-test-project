if (oField.name == "subdivision_id" || oField.name == "subdivision_text") {
    oField.value = getFormField(oField.name, "");
} else {
    oField.value = getFormField(oField.name, oField.value);
}
