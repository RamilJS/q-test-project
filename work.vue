// Выборка: меню портала (код меню приходит из LPE-параметра sMenuCode, сейчас = "header").
//
// ИСТОРИЯ (02.11.2026, после обновления платформы):
// 1. Симптом: в Network ответ выборки с MESSAGE "Текущий сайт не инициализирован", меню пустое.
// 2. Причина: после обновления выборка вызывается через /pp/Ext5/extjs_json_collection_data.html,
//    и переменная curSite в её контексте НЕ определена. Подтверждено логом diag_cur_site, запущенным
//    в коде этой выборки: curSite -> "curSite not defined"; в Request.Session.Env поля curSite тоже нет
//    ("Unknown object property: curSite").
//    Обращение к неопределённой переменной в этом движке кидает исключение, поэтому проверка
//    try { curSite; } catch (...) срабатывала всегда и делала return false до построения меню.
// 3. Зачем был нужен curSite: ТОЛЬКО для блока "СТАРЫЙ ВАРИАНТ" ниже — он собирал меню из
//    curSite.menus (документ сайта) вручную. Но результат этого блока НЕ попадал в ответ: сразу после
//    него стояла строка RESULT = [];, которая его стирала, и в ответ уходил только "РАБОЧИЙ ВАРИАНТ"
//    через tools_web.get_menu_items(). То есть curSite и до обновления на итоговое меню не влиял —
//    он был нужен только проверке и мёртвому блоку.
// 4. Подтверждено реальным тестом в коде выборки (diag_cur_site v2): tools_web.get_menu_items(sMenuCode, 2, 1, 10)
//    работает без curSite и вернул 10 пунктов 1-го уровня. После замены кода меню на сайте появилось (02.11.2026).
// 5. Старый код НЕ удалён, а закомментирован ниже (блоки "ЗАКОММЕНТИРОВАНО 02.11.2026"), чтобы можно было
//    вернуть его, если выяснится, что он был нужен. Чтобы вернуть ручную сборку из документа сайта
//    без curSite, сайт нужно открывать явно: tools.open_doc(<id сайта>).TopElem вместо curSite.

sLogName = 'header_menu_collection';
EnableLog(sLogName, true);
function alert(sInputObj) { LogEvent(sLogName, sInputObj); return sInputObj; }

// ===== ЗАКОММЕНТИРОВАНО 02.11.2026: проверка curSite (после обновления curSite в выборке не определён,
// ===== проверка всегда падала и прерывала выборку; см. п.2-3 в шапке) =====
// try
// {
// curSite;
//
// }
// catch(err)
// {
// RESULT = [];
// ERROR = 523;
// MESSAGE = "Текущий сайт не инициализирован";
// alert("RemoteCollection: libPortal: GetMenu: " + MESSAGE);
//
// return false;
// }
//
//
// //var oRes1 = tools.call_code_library_method("libPortal", "GetMenu", [ 'header', curSiteID, curSite, Env, bReturnMenyEntryWithBlankUrl, bGetMenuCodeFromUrl, sUrlParamNameMenuCode ]);
// //alert(tools.object_to_text(oRes1, 'json'))

// ===== ЗАКОММЕНТИРОВАНО 02.11.2026: СТАРЫЙ ВАРИАНТ — ручная сборка меню из curSite.menus.
// ===== Его RESULT всё равно стирался строкой RESULT = []; ниже, на ответ не влиял (см. п.3 в шапке) =====
// _menu = ArrayOptFind(curSite.menus, "This.id == 'header'");
// _menu_1_lvl = ArraySelect(_menu.documents, "This.parent_document_id == undefined || This.parent_document_id == '' || This.parent_document_id == null");
// RESULT = [];
// for (_elem in _menu_1_lvl) {
// _link = String(_elem.link_url != "" ? _elem.link_url : ("https://als-wt/_wt/" + _elem.document_id));
// if (_elem.link_url == "") {
// _elemDoc = tools.open_doc(_elem.document_id).TopElem;
// if (_elemDoc.attributes.is_link) {
// _link = String(_elemDoc.attributes.link_href)
// }
// }
// RESULT.push(
// {
// id: String(_elem.id),
// parent_id: String(_elem.parent_document_id),
// text: String(_elem.name),
// path: _link,
// icon: "",
// }
// )
// _menu_2_lvl = ArraySelect(_menu.documents, "This.parent_document_id == _elem.id");
// for (__elem in _menu_2_lvl) {
// __link = String(__elem.link_url != "" ? __elem.link_url : ("https://als-wt/_wt/" + __elem.document_id));
// if (__elem.link_url == "") {
// __elemDoc = tools.open_doc(__elem.document_id).TopElem;
// if (__elemDoc.attributes.is_link) {
// __link = String(__elemDoc.attributes.link_href)
// }
// }
// RESULT.push(
// {
// id: String(__elem.id),
// parent_id: String(__elem.parent_document_id),
// text: String(__elem.name),
// path: __link,
// icon: "",
// }
// )
// }
// }
// RESULT = [];   // <- эта строка стирала результат СТАРОГО ВАРИАНТА

// ===== РАБОЧИЙ ВАРИАНТ (был в исходном коде и раньше; без изменений логики, только обёрнут в try/catch) =====
RESULT = [];
ERROR = 0;
MESSAGE = "";
try
{
    Session = Request.Session;
    arrMenuItems = tools_web.get_menu_items(sMenuCode, 2, 1, 10);

    /*---------------------- Отрисовка первого уровня ------------------------*/
    _id = 0;
    for (oMenuItem in arrMenuItems)
    {
        _id++;
        RESULT.push(
        {
            id: String(_id),
            parent_id: '',
            text: String(oMenuItem.name),
            path: oMenuItem.link_href,
            icon: ""
        });

        _id_child = 0;
        for (oFirstLevelItem in oMenuItem.items)
        {
            _id_child++;
            RESULT.push(
            {
                id: String(_id + "_" + _id_child),
                parent_id: String(_id),
                text: String(oFirstLevelItem.name),
                path: oFirstLevelItem.link_href,
                icon: ""
            });
        }
    }
    /*------------------------------------------------------------------------*/
}
catch (_ex)
{
    RESULT = [];
    ERROR = 1;
    MESSAGE = ExtractUserError(_ex);
    try { alert("RemoteCollection: header menu: ОШИБКА: " + MESSAGE); } catch (_exLog) {}
}

// (хвост исходного файла, был закомментирован и раньше)
//RESULT = oRes.array;

//alert("RemoteCollection: libPortal: GetMenuWithCheckRule.bs: RESIULT:\r\n" + EncodeJson(RESULT))

//ERROR = oRes.error;
//MESSAGE = oRes.errorText;
//if(ERROR != 0) alert("ERROR: RemoteCollection: libPortal: GetMenuWithCheckRule.bs: " + MESSAGE)
