.breadcrumbs-container {
  width: 100% !important;
  max-width: 1100px; /* Дефолтная ширина как у карточки для 1373-1599px */
  margin: 0 auto 20px !important;
  justify-content: flex-start !important; /* Выравниваем по левому краю */
  padding-left: 0 !important;
}

/* Убираем лишний отступ слева у самих хлебных крошек */
.q-breadcrumbs.g-ml-lg {
  margin-left: 0 !important;
}

/* Синхронизируем ширину с медиа-запросами карточки */
@media (min-width: 1920px) {
  .breadcrumbs-container {
    max-width: 1350px;
  }
}

@media (min-width: 1600px) and (max-width: 1919px) {
  .breadcrumbs-container {
    max-width: 1300px;
  }
}

@media (min-width: 1373px) and (max-width: 1599px) {
  .breadcrumbs-container {
    max-width: 1100px;
  }
}

@media (max-width: 1372px) {
  .breadcrumbs-container {
    max-width: 940px;
  }
}
