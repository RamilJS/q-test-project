var mgr = null;

// func_manager — это массив, перебираем его
for (var fm in coll.func_managers.func_manager) {
    if (fm.is_native == 1) {
        mgr = fm;
        break;
    }
}

if (mgr == null) {
    alert("Руководитель с is_native=1 не найден");
    return;
}

alert("Manager ID: " + mgr.person_id);
alert("Manager Name: " + mgr.person_fullname);
