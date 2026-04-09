---
read_when:
    - Chcesz, aby promowanie pamięci uruchamiało się automatycznie
    - Chcesz zrozumieć, co robi każda faza dreaming
    - Chcesz dostroić konsolidację bez zaśmiecania `MEMORY.md`
summary: Konsolidacja pamięci w tle z fazami light, deep i REM oraz Dream Diary
title: Dreaming (eksperymentalne)
x-i18n:
    generated_at: "2026-04-09T01:27:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 26476eddb8260e1554098a6adbb069cf7f5e284cf2e09479c6d9d8f8b93280ef
    source_path: concepts/dreaming.md
    workflow: 15
---

# Dreaming (eksperymentalne)

Dreaming to system konsolidacji pamięci w tle w `memory-core`.
Pomaga OpenClaw przenosić silne krótkoterminowe sygnały do trwałej pamięci, jednocześnie
zachowując przejrzystość i możliwość przeglądu procesu.

Dreaming jest funkcją **opt-in** i domyślnie jest wyłączone.

## Co zapisuje dreaming

Dreaming utrzymuje dwa rodzaje danych wyjściowych:

- **Stan maszyny** w `memory/.dreams/` (magazyn przypomnień, sygnały faz, punkty kontrolne ingestii, blokady).
- **Dane czytelne dla człowieka** w `DREAMS.md` (lub istniejącym `dreams.md`) oraz opcjonalne pliki raportów faz w `memory/dreaming/<phase>/YYYY-MM-DD.md`.

Promowanie do pamięci długoterminowej nadal zapisuje wyłącznie do `MEMORY.md`.

## Model faz

Dreaming używa trzech współpracujących faz:

| Faza | Cel                                       | Trwały zapis      |
| ----- | ----------------------------------------- | ----------------- |
| Light | Sortowanie i przygotowanie ostatnich materiałów krótkoterminowych | Nie               |
| Deep  | Ocenianie i promowanie trwałych kandydatów | Tak (`MEMORY.md`) |
| REM   | Refleksja nad tematami i powracającymi ideami | Nie               |

Te fazy są wewnętrznymi szczegółami implementacji, a nie oddzielnymi
konfigurowanymi przez użytkownika „trybami”.

### Faza light

Faza light pobiera ostatnie dzienne sygnały pamięci i ślady przypomnień, usuwa duplikaty
i przygotowuje wiersze kandydatów.

- Odczytuje stan krótkoterminowych przypomnień, ostatnie dzienne pliki pamięci oraz zredagowane transkrypty sesji, gdy są dostępne.
- Zapisuje zarządzany blok `## Light Sleep`, gdy magazyn zawiera dane wyjściowe inline.
- Rejestruje sygnały wzmacniające na potrzeby późniejszego rankingu deep.
- Nigdy nie zapisuje do `MEMORY.md`.

### Faza deep

Faza deep decyduje, co staje się pamięcią długoterminową.

- Ustala ranking kandydatów przy użyciu ważonej punktacji i progów.
- Wymaga spełnienia wartości `minScore`, `minRecallCount` i `minUniqueQueries`.
- Przed zapisem ponownie pobiera fragmenty z aktywnych plików dziennych, więc nieaktualne/usunięte fragmenty są pomijane.
- Dopisuje promowane wpisy do `MEMORY.md`.
- Zapisuje podsumowanie `## Deep Sleep` do `DREAMS.md` i opcjonalnie zapisuje `memory/dreaming/deep/YYYY-MM-DD.md`.

### Faza REM

Faza REM wyodrębnia wzorce i sygnały refleksyjne.

- Buduje podsumowania tematów i refleksji na podstawie ostatnich krótkoterminowych śladów.
- Zapisuje zarządzany blok `## REM Sleep`, gdy magazyn zawiera dane wyjściowe inline.
- Rejestruje sygnały wzmacniające REM używane przez ranking deep.
- Nigdy nie zapisuje do `MEMORY.md`.

## Ingestia transkryptów sesji

Dreaming może pobierać zredagowane transkrypty sesji do korpusu dreaming. Gdy
transkrypty są dostępne, są przekazywane do fazy light razem z dziennymi
sygnałami pamięci i śladami przypomnień. Treści osobiste i wrażliwe są redagowane
przed ingestą.

## Dream Diary

Dreaming prowadzi także narracyjny **Dream Diary** w `DREAMS.md`.
Gdy po każdej fazie zgromadzi się wystarczająco dużo materiału, `memory-core` uruchamia w tle
podrzędny turn subagenta typu best-effort (przy użyciu domyślnego modelu runtime)
i dopisuje krótki wpis do dziennika.

Ten dziennik jest przeznaczony do odczytu przez człowieka w interfejsie Dreams, a nie jako źródło promocji.

Istnieje również ugruntowana ścieżka historycznego backfill dla przeglądu i odzyskiwania:

- `memory rem-harness --path ... --grounded` wyświetla podgląd ugruntowanych wpisów dziennika na podstawie historycznych notatek `YYYY-MM-DD.md`.
- `memory rem-backfill --path ...` zapisuje odwracalne ugruntowane wpisy dziennika do `DREAMS.md`.
- `memory rem-backfill --path ... --stage-short-term` przygotowuje ugruntowanych trwałych kandydatów w tym samym magazynie dowodów krótkoterminowych, którego używa już normalna faza deep.
- `memory rem-backfill --rollback` i `--rollback-short-term` usuwają te przygotowane artefakty backfill bez naruszania zwykłych wpisów dziennika ani aktywnych krótkoterminowych przypomnień.

Control UI udostępnia ten sam przepływ backfill/reset dziennika, dzięki czemu możesz sprawdzić
wyniki w scenie Dreams przed podjęciem decyzji, czy ugruntowani kandydaci
zasługują na promocję. Scena pokazuje też odrębny ugruntowany tor, aby było widać,
które przygotowane wpisy krótkoterminowe pochodzą z historycznego odtworzenia, które promowane
elementy były prowadzone przez grounded, oraz umożliwia wyczyszczenie tylko ugruntowanych przygotowanych wpisów bez
naruszania zwykłego aktywnego stanu krótkoterminowego.

## Sygnały rankingu deep

Ranking deep używa sześciu ważonych sygnałów bazowych oraz wzmocnienia fazowego:

| Sygnał              | Waga   | Opis                                              |
| ------------------- | ------ | ------------------------------------------------- |
| Częstotliwość       | 0.24   | Ile krótkoterminowych sygnałów zgromadził wpis    |
| Trafność            | 0.30   | Średnia jakość odzyskiwania dla wpisu             |
| Różnorodność zapytań | 0.15   | Odrębne konteksty zapytania/dnia, które go ujawniły |
| Aktualność          | 0.15   | Wynik świeżości z zanikiem w czasie               |
| Konsolidacja        | 0.10   | Siła nawrotów w wielu dniach                      |
| Bogactwo pojęciowe  | 0.06   | Gęstość tagów pojęciowych z fragmentu/ścieżki     |

Trafienia w fazach light i REM dodają niewielkie wzmocnienie z zanikiem aktualności z
`memory/.dreams/phase-signals.json`.

## Harmonogram

Po włączeniu `memory-core` automatycznie zarządza jednym zadaniem cron dla pełnego
przebiegu dreaming. Każdy przebieg uruchamia fazy po kolei: light -> REM -> deep.

Domyślne zachowanie kadencji:

| Ustawienie          | Domyślnie  |
| ------------------- | ---------- |
| `dreaming.frequency` | `0 3 * * *` |

## Szybki start

Włącz dreaming:

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

Włącz dreaming z niestandardową kadencją przebiegu:

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true,
            "timezone": "America/Los_Angeles",
            "frequency": "0 */6 * * *"
          }
        }
      }
    }
  }
}
```

## Komenda slash

```
/dreaming status
/dreaming on
/dreaming off
/dreaming help
```

## Przepływ pracy CLI

Użyj promowania przez CLI do podglądu lub ręcznego zastosowania:

```bash
openclaw memory promote
openclaw memory promote --apply
openclaw memory promote --limit 5
openclaw memory status --deep
```

Ręczne `memory promote` domyślnie używa progów fazy deep, chyba że zostaną nadpisane
flagami CLI.

Wyjaśnij, dlaczego konkretny kandydat zostałby lub nie zostałby promowany:

```bash
openclaw memory promote-explain "router vlan"
openclaw memory promote-explain "router vlan" --json
```

Wyświetl podgląd refleksji REM, prawd kandydatów i danych wyjściowych promocji deep bez
zapisywania czegokolwiek:

```bash
openclaw memory rem-harness
openclaw memory rem-harness --json
```

## Kluczowe wartości domyślne

Wszystkie ustawienia znajdują się w `plugins.entries.memory-core.config.dreaming`.

| Klucz      | Domyślnie  |
| ---------- | ---------- |
| `enabled`   | `false`     |
| `frequency` | `0 3 * * *` |

Polityka faz, progi i zachowanie magazynu są wewnętrznymi szczegółami implementacji
(nie są to ustawienia dostępne dla użytkownika).

Pełną listę kluczy znajdziesz w [Memory configuration reference](/pl/reference/memory-config#dreaming-experimental).

## Interfejs Dreams

Po włączeniu karta **Dreams** w Gateway pokazuje:

- bieżący stan włączenia dreaming
- stan na poziomie faz oraz obecność zarządzanego przebiegu
- liczby elementów krótkoterminowych, ugruntowanych, sygnałów i promowanych dzisiaj
- czas następnego zaplanowanego uruchomienia
- odrębny ugruntowany tor sceny dla przygotowanych wpisów historycznego odtworzenia
- rozwijany czytnik Dream Diary oparty na `doctor.memory.dreamDiary`

## Powiązane

- [Memory](/pl/concepts/memory)
- [Memory Search](/pl/concepts/memory-search)
- [memory CLI](/cli/memory)
- [Memory configuration reference](/pl/reference/memory-config)
