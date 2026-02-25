while (ArrayCount(oReqDocTE.persons) > 0)
{
    oReqDocTE.persons[0].Delete();
}


// очищаем существующих
if (oReqDocTE.persons != undefined)
{
    for (var i = ArrayCount(oReqDocTE.persons) - 1; i >= 0; i--)
    {
        oReqDocTE.persons.DeleteChildByIndex(i);
    }
}
