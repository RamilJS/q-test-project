// ДИАГНОСТИКА v2: что доступно в коде ВЫБОРКИ (вызов через /pp/Ext5/extjs_json_collection_data.html).
// Вставить в начало кода выборки (перед блоком try { curSite; }), открыть страницу с меню,
// прислать ВЕСЬ лог 'diag_cur_site' от START до END.
// v2: + дамп Request.Session.Env, Referer/Host, пробный вызов tools_web.get_menu_items().

sLogName = 'diag_cur_site';
EnableLog(sLogName, true);
function alert(sInputObj) { LogEvent(sLogName, sInputObj); return sInputObj; }

alert("==== START " + StrDate(Date(), true));

// --- переменные сайта/окружения ---
try { alert("curSite.id = " + String(curSite.id)); } catch (_e1) { alert("curSite: НЕТ (" + ExtractUserError(_e1) + ")"); }
try { alert("curSiteID = " + String(curSiteID)); } catch (_e2) { alert("curSiteID: НЕТ (" + ExtractUserError(_e2) + ")"); }
try { alert("Env = " + tools.object_to_text(Env, 'json')); } catch (_e3) { alert("Env: НЕТ (" + ExtractUserError(_e3) + ")"); }
try { alert("Env.curSiteID = " + String(Env.curSiteID)); } catch (_e4) { alert("Env.curSiteID: НЕТ (" + ExtractUserError(_e4) + ")"); }
try { alert("Env.curSite.id = " + String(Env.curSite.id)); } catch (_e5) { alert("Env.curSite: НЕТ (" + ExtractUserError(_e5) + ")"); }
try { alert("Request.Session.Env.curSiteID = " + String(Request.Session.Env.curSiteID)); } catch (_e6) { alert("Request.Session.Env.curSiteID: НЕТ (" + ExtractUserError(_e6) + ")"); }
try { alert("Request.Session.Env.curSite.id = " + String(Request.Session.Env.curSite.id)); } catch (_e7) { alert("Request.Session.Env.curSite: НЕТ (" + ExtractUserError(_e7) + ")"); }
try { alert("curUserID = " + String(curUserID)); } catch (_e8) { alert("curUserID: НЕТ (" + ExtractUserError(_e8) + ")"); }
try { alert("sMenuCode = " + String(sMenuCode)); } catch (_e10) { alert("sMenuCode: НЕТ (" + ExtractUserError(_e10) + ")"); }

// --- полный дамп Env сессии: ищем id сайта под другим именем ---
try { alert("DUMP Request.Session.Env = " + tools.object_to_text(Request.Session.Env, 'json')); } catch (_e12) { alert("DUMP Request.Session.Env: ОШИБКА (" + ExtractUserError(_e12) + ")"); }

// --- откуда пришёл запрос ---
try { alert("Request.Url = " + String(Request.Url)); } catch (_e9) { alert("Request.Url: НЕТ (" + ExtractUserError(_e9) + ")"); }
try { alert("Referer = " + String(Request.Header.GetOptProperty("Referer"))); } catch (_e13) { alert("Referer: ОШИБКА (" + ExtractUserError(_e13) + ")"); }
try { alert("Host = " + String(Request.Header.GetOptProperty("Host"))); } catch (_e14) { alert("Host: ОШИБКА (" + ExtractUserError(_e14) + ")"); }

// --- пробный вызов: работает ли tools_web.get_menu_items без curSite ---
try
{
    Session = Request.Session;
    arrDiagMenu = tools_web.get_menu_items(sMenuCode, 2, 1, 10);
    alert("get_menu_items: OK, пунктов 1-го уровня = " + ArrayCount(arrDiagMenu));
    for (oDiagItem in arrDiagMenu)
    {
        alert("  menu: " + String(oDiagItem.name) + " -> " + String(oDiagItem.link_href));
    }
}
catch (_e15) { alert("get_menu_items: ОШИБКА (" + ExtractUserError(_e15) + ")"); }

// --- список сайтов ---
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
