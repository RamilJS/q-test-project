.event-item {
  border-radius: 12px;
  transition: background .2s ease,
              transform .2s ease,
              box-shadow .2s ease;
}

.event-item:hover {
  background: rgba(255,255,255,.12);
  transform: translateY(-2px);
  box-shadow: 0 6px 18px rgba(0,0,0,.25);
}

.event-item .q-icon {
  transition: transform .2s ease;
}

.event-item:hover .q-icon {
  transform: translateX(5px);
}
