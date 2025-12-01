var d = Trim(getFormField("duties", ""));
oReqDocTE.custom_elems.ObtainChildByKey("f_candidate_function").value =
    d != "" ? d : Trim(getFormField("duties_history", ""));
