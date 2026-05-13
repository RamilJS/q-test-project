copy(
[...document.querySelectorAll('.WssListRow')]
.map(r =>
  [...r.querySelectorAll('.WssListCell')]
  .map(c => c.innerText.trim())
  .join(' | ')
).join('\n')
)
