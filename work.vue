else
{
    var objectsRaw = annals.au.history.objects;
    var objectsArr = [];

    // 🔥 нормализация структуры
    if (ArrayIsArray(objectsRaw)) {
        objectsArr = objectsRaw;
    }
    else if (objectsRaw.object != undefined) {
        if (ArrayIsArray(objectsRaw.object))
            objectsArr = objectsRaw.object;
        else
            objectsArr = [objectsRaw.object];
    }
    else {
        objectsArr = [objectsRaw];
    }

    alert("objectsArr count = " + ArrayCount(objectsArr));
