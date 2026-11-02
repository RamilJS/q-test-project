// ДИАГНОСТИКА: какие переменные окружения (сайт/сессия) реально доступны в выборке после обновления.
// Вставить ВРЕМЕННО в начало выборки (вместо блока try { curSite; } catch ...), открыть страницу,
// затем посмотреть лог 'diag_cur_site' на сервере.

sLogName = 'diag_cur_site';
EnableLog(sLogName, true);
function alert(sInputObj) { LogEvent(sLogName, sInputObj); return sInputObj; }

alert("==== START " + StrDate(Date(), true));

try { alert("curSite.id = " + String(curSite.id)); } catch (_e1) { alert("curSite: НЕТ (" + ExtractUserError(_e1) + ")"); }
try { alert("curSiteID = " + String(curSiteID)); } catch (_e2) { alert("curSiteID: НЕТ (" + ExtractUserError(_e2) + ")"); }
try { alert("Env = " + tools.object_to_text(Env, 'json')); } catch (_e3) { alert("Env: НЕТ (" + ExtractUserError(_e3) + ")"); }
try { alert("Env.curSiteID = " + String(Env.curSiteID)); } catch (_e4) { alert("Env.curSiteID: НЕТ (" + ExtractUserError(_e4) + ")"); }
try { alert("Env.curSite.id = " + String(Env.curSite.id)); } catch (_e5) { alert("Env.curSite: НЕТ (" + ExtractUserError(_e5) + ")"); }
try { alert("Session.Env.curSiteID = " + String(Request.Session.Env.curSiteID)); } catch (_e6) { alert("Request.Session.Env.curSiteID: НЕТ (" + ExtractUserError(_e6) + ")"); }
try { alert("Session.Env.curSite.id = " + String(Request.Session.Env.curSite.id)); } catch (_e7) { alert("Request.Session.Env.curSite: НЕТ (" + ExtractUserError(_e7) + ")"); }
try { alert("curUserID = " + String(curUserID)); } catch (_e8) { alert("curUserID: НЕТ (" + ExtractUserError(_e8) + ")"); }
try { alert("Request.Url = " + String(Request.Url)); } catch (_e9) { alert("Request.Url: НЕТ (" + ExtractUserError(_e9) + ")"); }
try { alert("sMenuCode = " + String(sMenuCode)); } catch (_e10) { alert("sMenuCode: НЕТ (" + ExtractUserError(_e10) + ")"); }

// Список сайтов в системе — чтобы знать id нужного сайта для фолбэка
try
{
    arrSites = ArraySelectAll(XQuery("for $elem in sites return $elem"));
    for (oSite in arrSites)
    {
        alert("site: id=" + String(oSite.id) + " | name=" + String(oSite.name));
    }
}
catch (_e11) { alert("XQuery sites: ОШИБКА (" + ExtractUserError(_e11) + ")"); }

alert("==== END");
RESULT = [];
