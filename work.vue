Отличное решение от тимлида — ровно та безопасная версия автоматизации, которую мы и обсуждали: не трогаем существующее (не переименовывая суть), не удаляем ничего, дозаполнение translate/owner_id остаётся за человеком. Вот как это будет выглядеть в коде агента «АДМИН. ИНТЕГРАЦИЯ».

1. Существующую функцию просто переименовать, тело менять не нужно:
function UpdateMirCodesForCollaborators(){
    // ...весь код, который сейчас лежит в UpdateMirCodes(), без изменений...
}

2. Новая функция UpdateMirCodes() — сравнивает коды из 1С с каталогом и добавляет недостающие:
function UpdateMirCodes(){
    //сравнивает коды МИР, реально пришедшие из 1С (globalObjects.array.mir_codes),
    //со справочником cc_mir_codes и добавляет туда те, которых ещё нет.
    //старые коды из справочника не удаляются -- они могут понадобиться в будущем.
    //у новых кодов заполняется только name, остальные поля (translate, owner_id)
    //остаются пустыми -- их дозаполняет человек вручную через существующий редактор.
    var existingMirCodeNames, existingMirCodeElem, mirCodeGroup, mirCodePair, newMirCodeName, newMirCodeNamesArray, tMirCodeDoc;

    existingMirCodeNames = [];
    for (existingMirCodeElem in XQuery('cc_mir_codes')) {
        existingMirCodeNames.push(StrUpperCase(Trim(String(existingMirCodeElem.name))));
    }

    newMirCodeNamesArray = [];
    for (mirCodeGroup in globalObjects.array.mir_codes) {
        for (mirCodePair in mirCodeGroup.array) {
            newMirCodeName = StrUpperCase(Trim(String(mirCodePair.MirCode)));
            if (newMirCodeName != ""
                && ArrayOptFind(existingMirCodeNames, "This == newMirCodeName") == undefined
                && ArrayOptFind(newMirCodeNamesArray, "This == newMirCodeName") == undefined) {
                newMirCodeNamesArray.push(newMirCodeName);
            }
        }
    }

    alert("UpdateMirCodes(). Новых кодов МИР найдено: " + ArrayCount(newMirCodeNamesArray));

    for (newMirCodeName in newMirCodeNamesArray) {
        tMirCodeDoc = tools.new_doc_by_name("cc_mir_code");
        tMirCodeDoc.BindToDb(DefaultDb);
        tMirCodeDoc.TopElem.name = newMirCodeName;
        tMirCodeDoc.Save();
        alert("UpdateMirCodes(). Добавлен новый код МИР в каталог: " + newMirCodeName);
    }
}

Пара технических решений, которые я заложил, стоит подтвердить с тимлидом или просто иметь в виду:

Беру весь каталог одним вызовом XQuery('cc_mir_codes') (по тому же принципу, что и в ВТБЛ. Выборка Мир-Кодов) и сравниваю в памяти, а не хожу в БД по каждому коду отдельно — кодов там немного (~50), так быстрее и проще.
И новые коды из 1С, и уже существующие в каталоге я привожу к верхнему регистру и обрезаю пробелы (StrUpperCase(Trim(...))) перед сравнением — это на случай, если из 1С придёт код не в том регистре, что в справочнике, чтобы не наплодить дублей вроде LCOP и lcop. Заодно это совпадает с тем, как ручная форма редактирования сама приводит код к верхнему регистру при сохранении (StrUpperCase(getFormField("name", ...))), то есть автоматика и ручной ввод будут давать одинаковый результат по регистру.

3. Место вызова в конце скрипта — сейчас там один вызов, станет два:
if (globalObjects.update_flags.mir_codes) {
    alert('------------------------------------start update mir_codes------------------------------------');
    UpdateMirCodes();
    UpdateMirCodesForCollaborators();
    alert('------------------------------------end update mir_codes------------------------------------');
}

Порядок специально такой: сначала пополняем справочник (UpdateMirCodes), потом уже присваиваем коды сотрудникам (UpdateMirCodesForCollaborators) — функционально это не обязательно (они не зависят друг от друга), но логически правильнее сначала актуализировать источник правды, а потом уже раскладывать его по сотрудникам.

Готово к тому, чтобы вставить в реальный агент — если после первого тестового прогона (с TEST_TOP_N-аналогом или на предпроде, если там есть) увидишь что-то неожиданное в логах — присылай, разберём.
