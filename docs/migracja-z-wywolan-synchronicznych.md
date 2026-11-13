# Migracja z wywołań synchronicznych na zdarzenia Kafki

Stan na dziś: serwisy komunikują się przez `/internal/...` (HTTP, z ponawianiem). Ten dokument mapuje te wywołania na zdarzenia z `events/`. Kafka działa lokalnie (`rynek-infra/local`, `localhost:9092`), ale żaden serwis jej jeszcze nie używa.

## Mapa

| Obecne wywołanie `/internal` | Zdarzenie | Producent | Konsument |
|---|---|---|---|
| `POST /internal/orders/{id}/cancelled` (order -> payment) | `order.cancelled` | order-service | payment-service (zwrot pieniędzy za opłacone), catalog-service (zwolnienie zapasu, dziś osobne wywołanie `stock/release`), notification |
| `PUT /internal/orders/{id}/shipment` + `POST /internal/orders/{id}/shipment-status` z `PICKED_UP` (shipping -> order) | `order.shipped` | order-service (po przyjęciu zmiany statusu od shipping; źródłem jest `shipment.status-changed`) | notification, analytics |
| `POST /internal/orders/{id}/shipment-status` z `DELIVERED` (shipping -> order) | `order.fulfilled` | order-service | payment-service (saldo sprzedawcy dostępne po realizacji), engagement-service (można wystawić opinię), notification |
| zwrot przyjęty przez sprzedawcę (`/seller/orders/{id}/returns/...`), dziś wywołanie order -> payment oraz `POST /internal/orders/{id}/refunded` (payment -> order) | `return.accepted` | order-service | payment-service (wykonanie zwrotu); wynik wraca jako `payment.refunded` (poza zakresem tego etapu) |
| `PUT /internal/users/{id}/channel` (channel -> identity) | `channel.created` | channel-service | identity-service (przypisanie kanału do konta), search, feed |
| `PUT /internal/orders/{id}/reviewed` i `PUT /internal/bookings/{id}/reviewed` (engagement -> order/booking) | `review.created` | engagement-service | order-service i booking-service (oznaczenie „oceniono”, wg `targetType`), channel-service (ocena), search, notification |
| `POST /internal/bookings/{id}/cancelled` (booking -> payment) | `booking.cancelled` | booking-service | payment-service (zwrot zaliczki), notification, analytics |
| `POST /internal/orders/{id}/mark-paid` i `POST /internal/bookings/{id}/mark-paid` (payment -> order/booking) | `payment.paid` | payment-service | order-service (gdy `purpose=ORDER`), booking-service (gdy `purpose=BOOKING_DEPOSIT`), notification |

Nie migrujemy wywołań, które potrzebują odpowiedzi w tym samym żądaniu (to zostaje HTTP): odczyt cen i zapasu z catalog (`price-and-stock`, `stock/reserve`), koszyk dla order, dane kanału do kart ofert, `/internal/ratings`, `/internal/users/{id}` i adresy.

## Koperta zdarzenia

Każde zdarzenie ma: `eventId` (UUID, deduplikacja), `eventType` (nazwa zdarzenia i tematu), `occurredAt` (ISO-8601 UTC), `aggregateId` (UUID agregatu, klucz partycji) i `payload`. Schematy: `events/<nazwa>.schema.json` (JSON Schema 2020-12).

## Zasady wdrożenia (do zrobienia, osobno dla każdej pary)

1. **Kolejność:** najpierw konsument (obsługa zdarzenia, idempotentna po `eventId`), potem producent publikuje zdarzenie obok starego wywołania, na końcu usunięcie wywołania i klienta HTTP.
2. **Niezawodność publikacji:** zapis stanu i zdarzenia w jednej transakcji (outbox w bazie serwisu), a dopiero potem wysyłka do Kafki. Zastępuje dzisiejsze ponawianie z klientów HTTP.
3. **Dostarczanie:** at-least-once, więc konsumenci muszą być idempotentni (tabela przetworzonych `eventId` albo naturalna idempotencja stanu).
4. **Kolejność per agregat:** klucz wiadomości = `aggregateId`.
5. **Błędy:** po kilku nieudanych próbach wiadomość trafia na temat `<zdarzenie>.dlq`.
6. **Zmiany schematu:** niezgodne wstecz wymagają nowej wersji zdarzenia (np. `order.cancelled.v2`).

## Rozbieżności z §17 do rozstrzygnięcia

- `payment.paid` vs `payment.succeeded` z §17: wybrać jedną nazwę (propozycja: `payment.paid`, spójna z `mark-paid`).
- `order.fulfilled` vs `order.completed` z §17: dokument używa `order.completed`; tu `order.fulfilled`, bo status zamówienia to „Zrealizowane”.
- `return.accepted` nie ma odpowiednika w §17.
- `order.shipped` w §17 publikuje order, a `shipment.status-changed` shipping; tu order publikuje `order.shipped` dopiero po zmianie statusu.
- Konsumenci spoza obecnego kodu (search, feed, notification, analytics) wpisani zgodnie z §17; serwisy mogą być jeszcze szkieletami.
