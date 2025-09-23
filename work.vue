.guide-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.7); /* затемнение страницы */
  z-index: 2000;
  pointer-events: auto; /* оверлей блокирует клики */
}

/* Подсветка элемента */
.highlight-target {
  box-shadow: 0 4px 20px rgba(0,0,0,0.5), 0 0 0 4px black; /* черная рамка + тень */
  border-radius: 6px;
  position: relative;
  z-index: 2200; /* выше оверлея */
  background-color: white; /* чтобы текст не затемнялся через overlay */
}

/* Всплывающая подсказка */
.guide-tooltip {
  background: white;
  color: black;
  border-radius: 8px;
  padding: 16px;
  box-shadow: 0 4px 20px rgba(0,0,0,0.5);
  z-index: 2300;
  max-width: 300px;
}

