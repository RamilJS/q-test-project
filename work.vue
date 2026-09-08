// =====================================================================
// HREDU-183/181. Диагностика: точные системные имена типов документов для
// catalog у foreign_elem-полей (матрица обучения, типовая должность).
//
// Контекст: в модалке фильтров (HREDU-183_filtry_modal_shag1.js) поле "Мир-код"
// заработало с catalog: "cc_mir_code" -- подтвердило гипотезу, что foreign_elem.catalog
// это singular СИСТЕМНОЕ ИМЯ ТИПА ДОКУМЕНТА, а не название XQuery-коллекции во
// множественном числе. Но конкретные догадки "cc_learning_matrice" (для матрицы) и
// "common_position" (для типовой должности) оказались неверны -- дальше гадать не
// имеет смысла, есть надёжный способ узнать точно: у любого открытого документа есть
// системное свойство .Name (с большой буквы) -- это и есть тот самый внутренний
// идентификатор типа (см. в примере пользователя: "if (oEventTE.Name == 'education_method')").
//
// Что делает скрипт: открывает по ID уже известные нам документы -- одну запись
// матрицы (ID из недавнего теста, "Матрица тест 3") и документ "типовой должности"
// (ID 7679114597049004988, тот самый, который нашёлся в диагностике position_common_id
// в HREDU-181 -- он ссылается именно на объект типовой должности) -- и печатает их .Name.
//
// Как запустить: как тестовый remote_action/скрипт-агент. Пришли мне вывод alert() целиком.
// =====================================================================

function Run()
{
    var iMatrixID, iCommonPositionID, oDoc, sReport;

    iMatrixID = 7682761831139375285;         // "Матрица тест 3" -- из недавнего лога Run()
    iCommonPositionID = 7679114597049004988; // положение "типовая должность" -- из диагностики position_common_id

    sReport = "";

    try
    {
        oDoc = tools.open_doc(iMatrixID).TopElem;
        sReport = sReport + "Матрица (id=" + iMatrixID + "): .Name = [" + String(oDoc.Name) + "], .name (название) = [" + String(oDoc.name) + "]\r\n";
    }
    catch (_ex1)
    {
        sReport = sReport + "Матрица (id=" + iMatrixID + "): ОШИБКА открытия -- " + ExtractUserError(_ex1) + "\r\n";
    }

    try
    {
        oDoc = tools.open_doc(iCommonPositionID).TopElem;
        sReport = sReport + "Типовая должность (id=" + iCommonPositionID + "): .Name = [" + String(oDoc.Name) + "], .name (название) = [" + String(oDoc.name) + "]\r\n";
    }
    catch (_ex2)
    {
        sReport = sReport + "Типовая должность (id=" + iCommonPositionID + "): ОШИБКА открытия -- " + ExtractUserError(_ex2) + "\r\n";
    }

    alert("--- Системные имена типов документов ---\r\n" + sReport);
}

Run();
