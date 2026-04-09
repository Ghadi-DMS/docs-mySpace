---
read_when:
    - Chcesz zrozumieć, jak działa pamięć
    - Chcesz wiedzieć, które pliki pamięci zapisywać
summary: Jak OpenClaw zapamiętuje rzeczy między sesjami
title: Przegląd pamięci
x-i18n:
    generated_at: "2026-04-09T01:27:50Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2fe47910f5bf1c44be379e971c605f1cb3a29befcf2a7ee11fb3833cbe3b9059
    source_path: concepts/memory.md
    workflow: 15
---

# Przegląd pamięci

OpenClaw zapamiętuje rzeczy, zapisując **zwykłe pliki Markdown** w obszarze roboczym
Twojego agenta. Model „pamięta” tylko to, co zostanie zapisane na dysku — nie ma
żadnego ukrytego stanu.

## Jak to działa

Twój agent ma trzy pliki związane z pamięcią:

- **`MEMORY.md`** — pamięć długoterminowa. Trwałe fakty, preferencje i
  decyzje. Wczytywany na początku każdej sesji DM.
- **`memory/YYYY-MM-DD.md`** — codzienne notatki. Bieżący kontekst i obserwacje.
  Notatki z dziś i z wczoraj są wczytywane automatycznie.
- **`DREAMS.md`** (eksperymentalny, opcjonalny) — Dream Diary oraz podsumowania
  przebiegów dreaming do przeglądu przez człowieka, w tym ugruntowane wpisy
  historycznego backfill.

Te pliki znajdują się w obszarze roboczym agenta (domyślnie `~/.openclaw/workspace`).

<Tip>
Jeśli chcesz, aby Twój agent coś zapamiętał, po prostu go o to poproś: „Zapamiętaj, że
preferuję TypeScript.” Zapisze to do odpowiedniego pliku.
</Tip>

## Narzędzia pamięci

Agent ma dwa narzędzia do pracy z pamięcią:

- **`memory_search`** — znajduje odpowiednie notatki za pomocą wyszukiwania
  semantycznego, nawet gdy sformułowanie różni się od oryginału.
- **`memory_get`** — odczytuje określony plik pamięci lub zakres wierszy.

Oba narzędzia są dostarczane przez aktywny plugin pamięci (domyślnie: `memory-core`).

## Towarzyszący plugin Memory Wiki

Jeśli chcesz, aby trwała pamięć działała bardziej jak utrzymywana baza wiedzy niż
tylko surowe notatki, użyj dołączonego pluginu `memory-wiki`.

`memory-wiki` kompiluje trwałą wiedzę do wiki vault z:

- deterministyczną strukturą stron
- ustrukturyzowanymi twierdzeniami i dowodami
- śledzeniem sprzeczności i aktualności
- generowanymi pulpitami
- skompilowanymi skrótami dla konsumentów agent/runtime
- natywnymi dla wiki narzędziami, takimi jak `wiki_search`, `wiki_get`, `wiki_apply` i `wiki_lint`

Nie zastępuje aktywnego pluginu pamięci. Aktywny plugin pamięci nadal
odpowiada za przypominanie, promowanie i dreaming. `memory-wiki` dodaje
warstwę wiedzy bogatą w pochodzenie obok niego.

Zobacz [Memory Wiki](/pl/plugins/memory-wiki).

## Wyszukiwanie w pamięci

Gdy skonfigurowany jest dostawca embeddingów, `memory_search` używa **wyszukiwania
hybrydowego** — łączącego podobieństwo wektorowe (znaczenie semantyczne) z dopasowaniem
słów kluczowych (dokładne terminy, takie jak identyfikatory i symbole kodu).
Działa to od razu po skonfigurowaniu klucza API dla dowolnego obsługiwanego dostawcy.

<Info>
OpenClaw automatycznie wykrywa dostawcę embeddingów na podstawie dostępnych kluczy API. Jeśli
masz skonfigurowany klucz OpenAI, Gemini, Voyage lub Mistral, wyszukiwanie w pamięci
jest włączane automatycznie.
</Info>

Szczegółowe informacje o działaniu wyszukiwania, opcjach dostrajania i konfiguracji
dostawców znajdziesz w [Memory Search](/pl/concepts/memory-search).

## Backendy pamięci

<CardGroup cols={3}>
<Card title="Wbudowany (domyślny)" icon="database" href="/pl/concepts/memory-builtin">
Oparty na SQLite. Działa od razu z wyszukiwaniem słów kluczowych, podobieństwem wektorowym i
wyszukiwaniem hybrydowym. Bez dodatkowych zależności.
</Card>
<Card title="QMD" icon="search" href="/pl/concepts/memory-qmd">
Lokalny sidecar z rerankingiem, rozszerzaniem zapytań i możliwością indeksowania
katalogów poza obszarem roboczym.
</Card>
<Card title="Honcho" icon="brain" href="/pl/concepts/memory-honcho">
Natywna dla AI pamięć między sesjami z modelowaniem użytkownika, wyszukiwaniem semantycznym i
świadomością wielu agentów. Wymaga instalacji pluginu.
</Card>
</CardGroup>

## Warstwa wiki wiedzy

<CardGroup cols={1}>
<Card title="Memory Wiki" icon="book" href="/pl/plugins/memory-wiki">
Kompiluje trwałą pamięć do wiki vault bogatego w pochodzenie z twierdzeniami,
pulpitami, trybem mostu i przepływami pracy przyjaznymi dla Obsidian.
</Card>
</CardGroup>

## Automatyczne opróżnianie pamięci

Zanim [kompaktowanie](/pl/concepts/compaction) podsumuje Twoją rozmowę, OpenClaw
uruchamia cichą turę, która przypomina agentowi o zapisaniu ważnego kontekstu w plikach
pamięci. Jest to domyślnie włączone — nie musisz nic konfigurować.

<Tip>
Opróżnianie pamięci zapobiega utracie kontekstu podczas kompaktowania. Jeśli Twój agent ma
w rozmowie ważne fakty, które nie zostały jeszcze zapisane do pliku, zostaną one
zapisane automatycznie, zanim nastąpi podsumowanie.
</Tip>

## Dreaming (eksperymentalne)

Dreaming to opcjonalny przebieg konsolidacji pamięci w tle. Zbiera
sygnały krótkoterminowe, ocenia kandydatów i promuje do pamięci
długoterminowej (`MEMORY.md`) tylko elementy spełniające kryteria.

Został zaprojektowany tak, aby utrzymywać wysoki stosunek sygnału do szumu w pamięci długoterminowej:

- **Opt-in**: domyślnie wyłączone.
- **Harmonogram**: po włączeniu `memory-core` automatycznie zarządza jednym cyklicznym zadaniem cron
  dla pełnego przebiegu dreaming.
- **Progowe**: promocje muszą przejść progi wyniku, częstotliwości przypominania i
  różnorodności zapytań.
- **Możliwe do przeglądu**: podsumowania faz i wpisy dziennika są zapisywane do `DREAMS.md`
  do przeglądu przez człowieka.

Informacje o zachowaniu faz, sygnałach oceniania i szczegółach Dream Diary znajdziesz w
[Dreaming (experimental)](/pl/concepts/dreaming).

## Ugruntowany backfill i promocja na żywo

System dreaming ma teraz dwa ściśle powiązane tory przeglądu:

- **Live dreaming** działa na podstawie krótkoterminowego magazynu dreaming w
  `memory/.dreams/` i to właśnie z niego korzysta normalna głęboka faza przy podejmowaniu decyzji, co
  może zostać przeniesione do `MEMORY.md`.
- **Grounded backfill** odczytuje historyczne notatki `memory/YYYY-MM-DD.md` jako
  samodzielne pliki dzienne i zapisuje ustrukturyzowany wynik przeglądu do `DREAMS.md`.

Grounded backfill jest przydatny, gdy chcesz ponownie przeanalizować starsze notatki i sprawdzić,
co system uważa za trwałe, bez ręcznego edytowania `MEMORY.md`.

Gdy użyjesz:

```bash
openclaw memory rem-backfill --path ./memory --stage-short-term
```

ugruntowani kandydaci trwałej pamięci nie są promowani bezpośrednio. Są etapowani do
tego samego krótkoterminowego magazynu dreaming, z którego korzysta już normalna głęboka faza. To
oznacza, że:

- `DREAMS.md` pozostaje powierzchnią przeglądu dla człowieka.
- magazyn krótkoterminowy pozostaje powierzchnią rankingową dla maszyn.
- `MEMORY.md` nadal jest zapisywany wyłącznie przez głęboką promocję.

Jeśli uznasz, że ponowne odtworzenie nie było przydatne, możesz usunąć etapowane artefakty
bez naruszania zwykłych wpisów dziennika ani normalnego stanu przypominania:

```bash
openclaw memory rem-backfill --rollback
openclaw memory rem-backfill --rollback-short-term
```

## CLI

```bash
openclaw memory status          # Sprawdź stan indeksu i dostawcę
openclaw memory search "query"  # Wyszukiwanie z wiersza poleceń
openclaw memory index --force   # Odbuduj indeks
```

## Dalsza lektura

- [Builtin Memory Engine](/pl/concepts/memory-builtin) — domyślny backend SQLite
- [QMD Memory Engine](/pl/concepts/memory-qmd) — zaawansowany lokalny sidecar
- [Honcho Memory](/pl/concepts/memory-honcho) — natywna dla AI pamięć między sesjami
- [Memory Wiki](/pl/plugins/memory-wiki) — skompilowany sejf wiedzy i narzędzia natywne dla wiki
- [Memory Search](/pl/concepts/memory-search) — potok wyszukiwania, dostawcy i
  dostrajanie
- [Dreaming (experimental)](/pl/concepts/dreaming) — promocja w tle
  z krótkoterminowego przypominania do pamięci długoterminowej
- [Memory configuration reference](/pl/reference/memory-config) — wszystkie opcje konfiguracji
- [Compaction](/pl/concepts/compaction) — jak kompaktowanie współdziała z pamięcią
