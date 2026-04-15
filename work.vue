// --- DEBUG ---
alert("ObjectType(history) = " + ObjectType(history));
alert("ObjectType(history.objects) = " + ObjectType(history.objects));
alert("ArrayCount(history.objects) = " + ArrayCount(history.objects));

var firstObj = ArrayFirstElem(history.objects);
alert("firstObj = " + firstObj);
alert("ObjectType(firstObj) = " + ObjectType(firstObj));
alert("firstObj.section = " + firstObj.section);
alert("ObjectType(firstObj.section) = " + ObjectType(firstObj.section));
// --- END DEBUG ---
