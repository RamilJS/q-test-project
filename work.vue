
EnableLog('HREDU-183_7683878110140100214', true);
function diagAlert(_string) {
    LogEvent('HREDU-183_7683878110140100214', _string);
    return _string;
}
function Run()
{
    diagAlert("проверка -- эта функция называется НЕ alert(), а diagAlert()");
    RESULT = [{ fullname: "ТЕСТ БИСЕКЦИИ (переименованный alert)", position_name: "-", subdivision_name: "-", macroregion: "-", city: "-", program_name: "-", completion_date: "", in_audience: true }];
}
Run();
