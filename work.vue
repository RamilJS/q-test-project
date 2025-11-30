<style>
.vtbl_tooltip .vtbl_tooltip_text {
    display: none; /* скрываем текст подсказки */
}

.vtbl_tooltip::after {
    content: '?';
    font-size: 18px;
    color: blue;
    cursor: pointer;
    margin-left: 5px;
}

.custom-tooltip {
    position: absolute;
    background: #00AAFF;
    color: #fff;
    padding: 6px 10px;
    border-radius: 6px;
    white-space: normal;
    max-width: 260px;
    z-index: 99999;
    font-size: 14px;
    box-shadow: 0 4px 10px rgba(0,0,0,0.25);
    pointer-events: none;
    transform: translate(-50%, -8px);
    opacity: 0;
    transition: opacity 0.15s ease;
}

.custom-tooltip::after {
    content: "";
    position: absolute;
    bottom: -6px;
    left: 50%;
    transform: translateX(-50%);
    border-width: 6px;
    border-style: solid;
    border-color: #00AAFF transparent transparent transparent;
}
</style>

<script>
$(document).ready(function() {

    let activeTooltip = null;

    function showTooltip(targetElem) {
        hideTooltip(); // скрываем старый tooltip, если есть

        let textElem = $(targetElem).find('.vtbl_tooltip_text');
        if (textElem.length === 0) return;

        let text = textElem.text().trim();
        if (!text) return;

        let tooltip = document.createElement("div");
        tooltip.className = "custom-tooltip";
        tooltip.innerText = text;
        document.body.appendChild(tooltip);

        // позиционируем tooltip над элементом
        let rect = targetElem.getBoundingClientRect();
        tooltip.style.left = (rect.left + rect.width / 2 + window.scrollX) + "px";
        tooltip.style.top = (rect.top - tooltip.offsetHeight - 6 + window.scrollY) + "px";

        // плавное появление
        requestAnimationFrame(() => {
            tooltip.style.opacity = 1;
        });

        activeTooltip = tooltip;
    }

    function hideTooltip() {
        if (activeTooltip) {
            activeTooltip.remove();
            activeTooltip = null;
        }
    }

    // показ тултипа при наведении
    $('body').on('mouseenter', '.vtbl_tooltip', function() {
        showTooltip(this);
    });

    // скрытие при уходе мыши
    $('body').on('mouseleave', '.vtbl_tooltip', function() {
        hideTooltip();
    });

});
</script>
