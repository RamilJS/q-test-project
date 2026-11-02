// Выборка: меню шапки портала (код меню приходит из LPE-параметра sMenuCode, сейчас = "header").
//
// ИСТОРИЯ (02.11.2026, после обновления платформы):
// - Было: в начале проверка try { curSite; } catch -> MESSAGE "Текущий сайт не инициализирован", return false.
//   После обновления выборка вызывается через /pp/Ext5/extjs_json_collection_data.html, и curSite / curSiteID
//   в её контексте НЕ определены (подтверждено логом diag_cur_site; в Request.Session.Env поля curSite тоже нет).
// - Первая половина старого кода (сборка меню из curSite.menus) на результат не влияла: её RESULT
//   затирался строкой RESULT = [] перед второй половиной. Удалена вместе с проверкой curSite.
// - Подтверждено реальным тестом в коде выборки (diag_cur_site v2): tools_web.get_menu_items(sMenuCode, 2, 1, 10)
//   работает без curSite и вернул 10 пунктов 1-го уровня. Оставлена только эта ветка.

sLogName = 'header_menu_collection';
EnableLog(sLogName, true);
function alert(sInputObj) { LogEvent(sLogName, sInputObj); return sInputObj; }

RESULT = [];
ERROR = 0;
MESSAGE = "";
try
{
    Session = Request.Session;
    arrMenuItems = tools_web.get_menu_items(sMenuCode, 2, 1, 10);

    /*---------------------- Первый уровень + дочерние ------------------------*/
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
    /*-------------------------------------------------------------------------*/
}
catch (_ex)
{
    RESULT = [];
    ERROR = 1;
    MESSAGE = ExtractUserError(_ex);
    try { alert("RemoteCollection: header menu: ОШИБКА: " + MESSAGE); } catch (_exLog) {}
}
