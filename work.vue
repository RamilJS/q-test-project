// --- ОЧИЩАЕМ существующих спонсоров ---
if (oReqDocTE.persons != undefined)
{
    while (oReqDocTE.persons.ChildNum > 0)
    {
        oReqDocTE.persons.DeleteChild(0);
    }
}
