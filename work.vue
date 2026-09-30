function SortRowsById(rows)
{
    return ArraySort(rows, "Int(This.id)", "+");
}
 

function BinarySearchById(sortedRows, targetId)
{
    var lo, hi, mid, midId, iTarget;
    iTarget = Int(targetId);
    lo = 0;
    hi = ArrayCount(sortedRows) - 1;
    while (lo <= hi)
    {
        mid = Int((lo + hi) / 2);
        midId = Int(sortedRows[mid].id);
        if (midId == iTarget) { return sortedRows[mid]; }
        else if (midId < iTarget) { lo = mid + 1; }
        else { hi = mid - 1; }
    }
    return undefined;
}
 

function SortIdArray(idArray)
{
    return ArraySort(idArray, "Int(This)", "+");
}
 

function IdArrayContainsSorted(sortedIdArray, value)
{
    var lo, hi, mid, midVal, iTarget;
    iTarget = Int(value);
    lo = 0;
    hi = ArrayCount(sortedIdArray) - 1;
    while (lo <= hi)
    {
        mid = Int((lo + hi) / 2);
        midVal = Int(sortedIdArray[mid]);
        if (midVal == iTarget) { return true; }
        else if (midVal < iTarget) { lo = mid + 1; }
        else { hi = mid - 1; }
    }
    return false;
}
 

function FindCity(sortedCityRows, collaboratorID)
{
    var cityRow, sCity;
    cityRow = BinarySearchById(sortedCityRows, collaboratorID);
    sCity = (cityRow != undefined && cityRow.sity != undefined ? String(cityRow.sity) : "");
    return (sCity != "" ? sCity : "(без города)");
}
 

function SortDateRowsByCollaboratorId(dateRows)
{
    return ArraySort(dateRows, "Int(This.collaborator_id)", "+");
}
 


function Run()
{
    RESULT = [{ fullname: "ТЕСТ БИСЕКЦИИ B", position_name: "-", subdivision_name: "-", macroregion: "-", city: "-", program_name: "-", completion_date: "", in_audience: true }];
}
Run();
