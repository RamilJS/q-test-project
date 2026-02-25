if (oReqDocTE.persons != undefined)
{
    for (var i = oReqDocTE.persons.ChildNum - 1; i >= 0; i--)
    {
        oReqDocTE.persons.RemoveChild(i);
    }
}
