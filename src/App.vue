import { date } from "quasar"

const formatXmlDate = (xmlDate) => {
  return date.formatDate(xmlDate, "DD.MM.YYYY HH:mm")
}

console.log(formatXmlDate("2011-12-26T10:45:15+00:00"))
// 👉 26.12.2011 10:45
