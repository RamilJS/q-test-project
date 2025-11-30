for (var i in coll.func_managers.func_manager) {
    var fm = coll.func_managers.func_manager[i];

    if (String(fm.is_native) == "1") {
        mgr = fm;
        break;
    }
}
