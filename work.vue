/* СТИЛЬ КОНТЕЙНЕРА ТУЛТИПА */
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
    transition: opacity .15s ease;
}

/* СТРЕЛОЧКА */
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

<script>

$(document).ready(function() {

    let activeTooltip = null;

    function showTooltip(elem, text) {
        hideTooltip();

        let tooltip = document.createElement("div");
        tooltip.className = "custom-tooltip";
        tooltip.innerText = text;

        document.body.appendChild(tooltip);

        let rect = elem.getBoundingClientRect();

        tooltip.style.left = (rect.left + rect.width / 2) + "px";
        tooltip.style.top = (rect.top - 10 + window.scrollY) + "px";

        // небольшая задержка чтобы сработала анимация opacity
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

    // Показ
    $('body').on('mouseenter', '.vtbl_tooltip', function() {
        let textElem = $(this).find('.vtbl_tooltip_text');
        if (textElem.length === 0) return;

        let text = textElem.text().trim();
        showTooltip(this, text);
    });

    // Скрытие
    $('body').on('mouseleave', '.vtbl_tooltip', function() {
        hideTooltip();
    });

});
</script>
