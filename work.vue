oForm.command = "close_form";
oForm.confirm_result = {
    command: "redirect",
    url: "/view_doc.html?" + Format("id=%1&matrix_id=%2&macroregion=%3&mir_code_id=%4&position_common_id=%5&program_id=%6",
        curObject.id, iMatrixID, sMacroregion, iMirCodeID, iPositionCommonID, iProgramID)
};
