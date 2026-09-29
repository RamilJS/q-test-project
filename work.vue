EnableLog('HREDU-183_7683878110140100214', true);
function alert(_string) {
    LogEvent('HREDU-183_7683878110140100214', _string);
    return _string;
}
 
 
DEBUG = true;              
LOG_NAME = "agent";        
CUR_OBJECT_ID = 7683878110140100214;         
 

function LogAlert(typeLog, message)
{
    
    
    try
    {
        tools.call_code_library_method("vtbl_log_lib", "LogAlert", [LOG_NAME, typeLog, CUR_OBJECT_ID, message, DEBUG]);
    }
    catch (_exLog)
    {
        
    }
}
 

PERF_DEBUG = true; 
gPerfStartTime = undefined;
gPerfLastTime = undefined;
 

function PerfStart()
{
    if (!PERF_DEBUG) { return; }
    gPerfStartTime = PerfNowSafe();
    gPerfLastTime = gPerfStartTime;
    PerfAlertSafe("[ЗАМЕР] СТАРТ. Время: " + PerfFormatTimestamp(gPerfStartTime));
}
 

function PerfCheckpoint(sLabel)
{
    if (!PERF_DEBUG) { return; }
    var dNow, sMsg;
    dNow = PerfNowSafe();
    sMsg = "[ЗАМЕР] " + sLabel + ". Время сейчас: " + PerfFormatTimestamp(dNow)
        + "; ЭТОТ шаг занял: " + PerfDiffSafe(gPerfLastTime, dNow)
        + "; всего с начала: " + PerfDiffSafe(gPerfStartTime, dNow);
    PerfAlertSafe(sMsg);
    gPerfLastTime = dNow;
}
 
function PerfNowSafe()
{
    try { return Date(); } catch (_ex) { return undefined; }
}
 
function PerfFormatTimestamp(dValue)
{
    try { return (dValue != undefined ? StrDate(dValue, true) : "?"); } catch (_ex) { return "?"; }
}
 
function PerfDiffSafe(dFrom, dTo)
{
    try
    {
        if (dFrom == undefined || dTo == undefined) { return "?"; }
        return String(Int((dTo - dFrom) * 86400)) + " сек";
    }
    catch (_ex)
    {
        return "? сек";
    }
}
 
function PerfAlertSafe(sMsg)
{
    try { alert(sMsg); } catch (_ex) {  }
}
 

function GetRequestUrlSafe()
{
    try
    {
        return String(Request.Url);
    }
    catch (_ex)
    {
        return "";
    }
}
 

function GetQueryParam(sUrl, sParamName)
{
    var sAmpMarker, sQMarkMarker, iParamPos, iValueStart, iAmpPos, iValueEnd, sRawValue, iUrlLen;
 
    iUrlLen = StrLen(sUrl);
 
    sAmpMarker = "&" + sParamName + "=";
    iParamPos = StrOptSubStrPos(sUrl, sAmpMarker, false);
    if (iParamPos != undefined)
    {
        iValueStart = iParamPos + StrLen(sAmpMarker);
    }
    else
    {
        sQMarkMarker = "?" + sParamName + "=";
        iParamPos = StrOptSubStrPos(sUrl, sQMarkMarker, false);
        if (iParamPos == undefined)
        {
            return "";
        }
        iValueStart = iParamPos + StrLen(sQMarkMarker);
    }
 
    iAmpPos = StrOptSubStrPos(sUrl, "&", false, iValueStart);
    iValueEnd = (iAmpPos != undefined ? iAmpPos : iUrlLen);
 
    sRawValue = StrRangePos(sUrl, iValueStart, iValueEnd);
 
    try
    {
        return UrlDecode(sRawValue);
    }
    catch (_exDecode)
    {
        return sRawValue;
    }
}
 

function GetActiveCollaboratorRows()
{
    LogAlert(1, "GetActiveCollaboratorRows(). НАЧАЛО");
    var collaboratorRows;
    collaboratorRows = ArraySelectAll(XQuery("for $elem in collaborators where $elem/is_dismiss=false() return $elem"));
    LogAlert(1, "GetActiveCollaboratorRows(). Найдено сотрудников: " + ArrayCount(collaboratorRows));
    LogAlert(1, "GetActiveCollaboratorRows(). КОНЕЦ");
    return collaboratorRows;
}
 

function GetPositionIdsByCommonPosition(iCommonPositionFilter)
{
    LogAlert(1, "GetPositionIdsByCommonPosition(). НАЧАЛО. iCommonPositionFilter=" + iCommonPositionFilter);
    var positionRows, positionIds, i;
    positionRows = ArraySelectAll(XQuery("for $elem in positions where $elem/position_common_id = " + iCommonPositionFilter + " return $elem"));
    positionIds = [];
    for (i = 0; i < ArrayCount(positionRows); i++)
    {
        positionIds.push(Int(positionRows[i].id));
    }
    LogAlert(1, "GetPositionIdsByCommonPosition(). Найдено конкретных должностей: " + ArrayCount(positionIds));
    LogAlert(1, "GetPositionIdsByCommonPosition(). КОНЕЦ");
    return positionIds;
}
 

function IdArrayContains(idArray, value)
{
    var i;
    for (i = 0; i < ArrayCount(idArray); i++)
    {
        if (Int(idArray[i]) == Int(value))
        {
            return true;
        }
    }
    return false;
}
 

function GetMacroregionRows()
{
    LogAlert(1, "GetMacroregionRows(). НАЧАЛО");
    var sqlText, macroRows;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_2ewj'']/value)[1]', 'varchar(max)') as macroregion\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    macroRows = ArraySelectAll(XQuery("sql:" + sqlText));
    LogAlert(1, "GetMacroregionRows(). Строк: " + ArrayCount(macroRows));
    LogAlert(1, "GetMacroregionRows(). КОНЕЦ");
    return macroRows;
}
 

function GetCityRows()
{
    LogAlert(1, "GetCityRows(). НАЧАЛО");
    var sqlText, rows;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''sity'']/value)[1]', 'varchar(max)') as sity\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    rows = ArraySelectAll(XQuery("sql:" + sqlText));
    LogAlert(1, "GetCityRows(). Строк: " + ArrayCount(rows));
    LogAlert(1, "GetCityRows(). КОНЕЦ");
    return rows;
}
 

function SortRowsById(rows)
{
    return ArraySort(rows, "Int(This.id)", "+");
}
 

function BinarySearchById(sortedRows, targetId)
{
    var lo, hi, mid, midId, iTarget;
    iTarget = Int(targetId);
    lo = 0;
    hi = ArrayCount(sortedRows) - 1;
    while (lo <= hi)
    {
        mid = Int((lo + hi) / 2);
        midId = Int(sortedRows[mid].id);
        if (midId == iTarget) { return sortedRows[mid]; }
        else if (midId < iTarget) { lo = mid + 1; }
        else { hi = mid - 1; }
    }
    return undefined;
}
 

function SortIdArray(idArray)
{
    return ArraySort(idArray, "Int(This)", "+");
}
 

function IdArrayContainsSorted(sortedIdArray, value)
{
    var lo, hi, mid, midVal, iTarget;
    iTarget = Int(value);
    lo = 0;
    hi = ArrayCount(sortedIdArray) - 1;
    while (lo <= hi)
    {
        mid = Int((lo + hi) / 2);
        midVal = Int(sortedIdArray[mid]);
        if (midVal == iTarget) { return true; }
        else if (midVal < iTarget) { lo = mid + 1; }
        else { hi = mid - 1; }
    }
    return false;
}
 

function FindCity(sortedCityRows, collaboratorID)
{
    var cityRow, sCity;
    cityRow = BinarySearchById(sortedCityRows, collaboratorID);
    sCity = (cityRow != undefined && cityRow.sity != undefined ? String(cityRow.sity) : "");
    return (sCity != "" ? sCity : "(без города)");
}
 

function SortDateRowsByCollaboratorId(dateRows)
{
    return ArraySort(dateRows, "Int(This.collaborator_id)", "+");
}
 

function GetMirCodeRows()
{
    LogAlert(1, "GetMirCodeRows(). НАЧАЛО");
    var sqlText, rows;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_mir_codes'']/value)[1]', 'varchar(max)') as mir_codes\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    rows = ArraySelectAll(XQuery("sql:" + sqlText));
    LogAlert(1, "GetMirCodeRows(). Строк: " + ArrayCount(rows));
    LogAlert(1, "GetMirCodeRows(). КОНЕЦ");
    return rows;
}
 

function ExtractMirCodes(rawValue)
{
    var parts, fields, codes, i;
    codes = [];
    parts = ArrayDirect(ArraySelect(String(rawValue).split("|"), "This != ''"));
    for (i = 0; i < ArrayCount(parts); i++)
    {
        fields = ArrayDirect(ArraySelect(String(parts[i]).split("#"), "This != ''"));
        if (ArrayCount(fields) > 0)
        {
            codes.push(String(fields[0]));
        }
    }
    return codes;
}
 

function CollaboratorHasMirCode(sortedMirCodeRows, collaboratorID, mirCodeFilter)
{
    var row, codes;
    row = BinarySearchById(sortedMirCodeRows, collaboratorID);
    if (row == undefined)
    {
        return false;
    }
    codes = ExtractMirCodes(row.mir_codes);
    return (ArrayOptFind(codes, "String(This) == String(mirCodeFilter)") != undefined);
}
 

function GetCompletionDateRows(programIds)
{
    LogAlert(1, "GetCompletionDateRows(). НАЧАЛО");
    var sqlText, dateRows;
    sqlText = "";
    sqlText = sqlText + "select ec.collaborator_id, e.education_method_id, min(ec.start_date) as first_date\r\n";
    sqlText = sqlText + "from event_collaborators ec\r\n";
    sqlText = sqlText + "join events e on e.id = ec.event_id\r\n";
    sqlText = sqlText + "where e.education_method_id in (" + ArrayMerge(programIds, "This", ",") + ")\r\n";
    sqlText = sqlText + "group by ec.collaborator_id, e.education_method_id";
    dateRows = ArraySelectAll(XQuery("sql:" + sqlText));
    LogAlert(1, "GetCompletionDateRows(). Строк: " + ArrayCount(dateRows));
    LogAlert(1, "GetCompletionDateRows(). КОНЕЦ");
    return dateRows;
}
 

function FindCompletionDate(sortedDateRows, collaboratorID, programID)
{
    var lo, hi, mid, midId, iTarget, iProgram, startIdx, i, n;
 
    iTarget = Int(collaboratorID);
    iProgram = Int(programID);
 
    
    lo = 0;
    hi = ArrayCount(sortedDateRows) - 1;
    startIdx = -1;
    while (lo <= hi)
    {
        mid = Int((lo + hi) / 2);
        midId = Int(sortedDateRows[mid].collaborator_id);
        if (midId == iTarget)
        {
            startIdx = mid;
            hi = mid - 1; 
        }
        else if (midId < iTarget) { lo = mid + 1; }
        else { hi = mid - 1; }
    }
 
    if (startIdx == -1) { return ""; }
 
    n = ArrayCount(sortedDateRows);
    for (i = startIdx; i < n && Int(sortedDateRows[i].collaborator_id) == iTarget; i++)
    {
        if (Int(sortedDateRows[i].education_method_id) == iProgram)
        {
            return StrDate(Date(sortedDateRows[i].first_date), false);
        }
    }
    return "";
}
 

function FindMacroregion(sortedMacroRows, collaboratorID)
{
    var macroRow;
    macroRow = BinarySearchById(sortedMacroRows, collaboratorID);
    return (macroRow != undefined && macroRow.macroregion != undefined ? String(macroRow.macroregion) : "");
}
 

function Run()
{
    LogAlert(1, "Run() -- БИСЕКЦИОННЫЙ ТЕСТ HREDU-183: все 5 НОВЫХ/уникальных функций (GetProgramIds/CollaboratorMatchesMatrixAudience/FindProgramName/ResolveMatrixContext/BuildReportRows) и их вызовы ВЫРЕЗАНЫ. Осталось только то, что byte-identical скопировано из проверенного рабочего файла HREDU-182, плюс шапка/логирование этого файла.");
    RESULT = [{ fullname: "ТЕСТ БИСЕКЦИИ HREDU-183", position_name: "-", subdivision_name: "-", macroregion: "-", city: "-", program_name: "-", completion_date: "", in_audience: true }];
}

 
Run();
