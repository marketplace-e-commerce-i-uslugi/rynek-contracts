# rynek-contracts

Wspólne kontrakty platformy **Rynek**: źródło prawdy dla interfejsów między serwisami i frontendem.

> Status: **nie rozpoczęty**. Są tylko struktura i mapa zdarzeń.

## Struktura

- `openapi/`: jedna specyfikacja OpenAPI 3 na serwis, np. `openapi/catalog-service.yaml`. Opisuje endpointy publiczne i `/internal/...`.
- `events/`: schematy zdarzeń Kafka. `events/README.md` zawiera mapę „kto publikuje, kto słucha”.

## Zasady

- Konwencje wspólne z §1 dokumentu architektury: prefiks `/api/v1`, paginacja kursorowa `{items, nextCursor}`, kwoty `{amount, currency}` w groszach, błędy RFC 7807, `Idempotency-Key`.
- Zmiana kontraktu najpierw trafia tutaj (PR), potem do serwisu.
- Zmiany niezgodne wstecz wymagają nowej wersji: `/api/v2` albo nowej wersji zdarzenia.
