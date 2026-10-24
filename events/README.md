# Zdarzenia Kafka

Konwencja nazw: `domena.zdarzenie` (np. `offer.published`). Schemat każdego zdarzenia trafi tu jako osobny plik (JSON Schema lub Avro, do decyzji).

## Mapa zdarzeń (§17 dokumentu architektury)

| Zdarzenie | Publikuje | Słuchają |
|---|---|---|
| `user.registered`, `user.roles-changed` | identity | channel, notification |
| `channel.created`, `channel.updated` | channel | search, feed |
| `channel.followed`, `channel.unfollowed` | channel | feed, notification, analytics |
| `offer.published`, `offer.updated`, `offer.unpublished` | catalog | search, feed, cart, analytics |
| `video.synced`, `video.attached`, `video.removed` | tiktok-integration | catalog, feed, search |
| `question.created`, `question.answered` | engagement | notification |
| `review.created` | engagement | channel (ocena), search, notification |
| `chat.message-sent` | chat | notification |
| `order.created`, `order.paid`, `order.shipped`, `order.completed`, `order.cancelled` | order | catalog (zapas), notification, analytics, engagement |
| `payment.succeeded`, `payment.failed`, `payment.refunded` | payment | order, booking, notification |
| `booking.created`, `booking.confirmed`, `booking.cancelled`, `booking.completed` | booking | notification, analytics, engagement |
| `shipment.status-changed` | shipping | order, notification |
