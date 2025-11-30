<style>

.wt-lp-tab-list-top {
    justify-content: space-between;
    align-items: stretch;
    background: radial-gradient(68% 100% at 100% 0%, rgb(119, 41, 255), rgba(116, 42, 245, 0)), radial-gradient(100% 98% at 100% 100%, rgb(4, 165, 236), rgba(41, 102, 227, 0)), linear-gradient(0deg, rgb(51, 90, 228), rgb(16, 25, 131));
    border-top-right-radius: 0.8rem;
    border-top-left-radius: 0.8rem;
}

.wt-lp-tab-item[wt-state='selected']{
    background: rgba(255,255,255,0.3)!important;
    color: #ffffff !important;
}

.wt-lp-tab-item:not([wt-state='selected']):hover {
    color: #ffffff !important;
    background-color: transparent !important;
    text-decoration: underline;
}

.wt-lp-macro-workarea {
    border-color: #CCCCCC;
    border-width: 1px;
    box-shadow: 0 0 1rem rgba(0, 0, 0, 0.5);
    border-radius: 0.8rem;
}

.wt-lp-wbutton-btn {
    background-color: transparent;
    background: linear-gradient(to right, #7a31f5, #578bf8) !important;
    transition: none !important;
    padding: 8px 16px !important;
}

.wt-lp-witemlist-link {
    padding: 0px;
    background: linear-gradient(to right, #7a31f5, #578bf8) !important;
    border-radius: 4px;
    font-family: Roboto, -apple-system, Helvetica Neue, Helvetica, Arial, sans-serif !important;
}

.wt-lp-witemlist-link:hover {
    background: #00aaff!important;
}

.wt-lp-witemlist-btn1-outer , .wt-lp-witemlist-btn1-outer:hover{
    background: transparent !important;
}

.vtbl_tooltip .vtbl_tooltip_text {
    display: none;
}

.vtbl_tooltip::after {
    content: '?';
    font-size: 18px;
    color: blue;
    cursor: pointer;
    margin-left: 5px;
}

.popper, .tooltip{
    z-index: 40001;
    background: #00AAFF;
}

.tooltip-inner {
     text-align: left !important;
     background-color: #00AAFF !important;
     color: #fff !important;
}

.tooltip {
    background-color: #00AAFF !important;
}

.tooltip .tooltip-arrow,
.tooltip.bs-tooltip-top .tooltip-arrow::before,
.tooltip[x-placement="top"] .tooltip-arrow::before {
    border-top-color: #00AAFF !important;
}

.tooltip.bs-tooltip-top .tooltip-arrow::after,
.tooltip[x-placement="top"] .tooltip-arrow::after {
    border-top-color: #00AAFF !important;
}

.tooltip .tooltip-arrow, .popper .popper__arrow {
    border-color: #00AAFF !important;
}

</style>

<script>

$(document).ready(function() {

    // --- ГЛАВНОЕ: прячем tooltip при уходе мыши ---
    $('body').on('mouseleave', '.vtbl_tooltip', function(e) {
        if (this._tooltipInstance) {
            this._tooltipInstance.hide();
        }
    });
    // -------------------------------------------------

    $('body').on('mouseenter', '.vtbl_tooltip', function(e) {

        let targetElem = this;
        let $targetElem = $(targetElem)

        if($(targetElem).prop("has_tooltip") != "1") {

            let $tooltipSpan = $(targetElem).find('.vtbl_tooltip_text');
            if ($tooltipSpan.length > 0) {

                let text = $tooltipSpan.text().trim();

                function createTooltip() {
                    // --- ВАЖНО: trigger: 'manual' ---
                    var tooltip = new Tooltip(targetElem, {
                        title: text,
                        placement: "top",
                        trigger: "manual",
                        container: "body"
                    });

                    tooltip.show();
                    targetElem._tooltipInstance = tooltip;
                }

                if(window.Popper==null){

                    $(document.createElement("link"))
                        .attr({ "wt-type": "lp-calendar", rel: "stylesheet", href: "/lp/fullcalendar/lib/tooltipe.css"})
                        .appendTo("head");

                    $.getScript("/lp/fullcalendar/lib/popper.min.js", function (){
                        $.getScript("/lp/fullcalendar/lib/tooltip.min.js", function (){
                            createTooltip();
                        });
                    });

                } else {
                    createTooltip();
                }
            }

            $(targetElem).prop("has_tooltip", "1");
        } else {
            // если уже создан – просто покажем снова
            if (targetElem._tooltipInstance) {
                targetElem._tooltipInstance.show();
            }
        }

    });

});
</script>
