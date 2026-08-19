# Messenger Links → plik + TickTick (Tasker)

Profil Taskera, który nasłuchuje na powiadomienia Messengera, zapisuje
wykryte linki do pliku na telefonie, a jeśli wiadomość jest od
**Paulina Kuczkowska** — dodatkowo tworzy przypomnienie w TickTick z tym
linkiem.

## Co zmieniło się względem oryginału

- **Profil** (`Messenger Links`) już nie jest ograniczony samym tytułem
  powiadomienia do „Paulina Kuczkowska” — teraz reaguje na powiadomienia
  z Messengera od **dowolnej osoby**, ale nadal tylko wtedy, gdy treść
  zawiera link (warunek regex w akcji If bez zmian).
- **Task** (`Messenger Links To Tasks`) ma nowe akcje:
  1. `If` (regex linku) — bez zmian.
  2. `Matches Regex` → wyciąga link do `%links` — bez zmian.
  3. **`Write File`** (nowe) — dopisuje linię z datą, nadawcą i linkiem do
     pliku `messenger_links.txt`.
  4. **`If %evtprm2 = "Paulina Kuczkowska"`** (nowe) — dalsze kroki tylko
     dla tej osoby.
  5. **`HTTP Request`** (nowe) — POST do TickTick Open API, tworzy zadanie
     z linkiem w treści.
  6. `End If`
  7. `Notify` — powiadomienie na telefonie, jak wcześniej (teraz dla
     każdego nadawcy, nie tylko Pauliny).
  8. `End If`

## Import do Taskera

1. Skopiuj plik `messenger_links.tasker.xml` na telefon (np. przez Google
   Drive, e-mail do siebie, albo `adb push`).
2. W Taskerze: **Profile → długie przytrzymanie w pustym miejscu → Import
   Profile** i wskaż plik (albo otwórz plik menedżerem plików — Tasker
   powinien zaproponować import).
3. Zaakceptuj uprawnienia, o które poprosi Tasker (dostęp do powiadomień,
   dostęp do pamięci).

## Krok 1: Zapis do pliku

Domyślna ścieżka to:

```
/storage/emulated/0/Documents/messenger_links.txt
```

Możesz ją zmienić w akcji **Write File** wewnątrz tasku. Na Androidzie
11+ może być potrzebne przyznanie Taskerowi uprawnienia „Zarządzanie
wszystkimi plikami” (All files access) w ustawieniach systemowych, żeby
zapis się udał.

Każda linia w pliku ma format:

```
GG:MM | Nadawca | https://link...
```

## Krok 2: TickTick — token API (wymagane ręcznie)

Tasker nie ma wbudowanej integracji z TickTick, więc używamy akcji
**HTTP Request** wywołującej oficjalne TickTick Open API.

1. Wejdź na https://developer.ticktick.com/manage i zarejestruj
   aplikację (Client ID/Secret) — potrzebne do przeprowadzenia
   autoryzacji OAuth2.
2. Przeprowadź przepływ OAuth2 (authorization code), żeby uzyskać
   **access token**. TickTick nie udostępnia tokenów długoterminowych z
   poziomu samej apki, więc token trzeba wygenerować raz przez OAuth
   (np. lokalnym skryptem/Postmanem) i potem odświeżać zgodnie z ich API.
3. W Taskerze utwórz zmienną globalną `%ticktick_token` z wartością
   tokena: **Profile → długie przytrzymanie → Task Variables** albo po
   prostu akcją `Variable Set` uruchomioną raz ręcznie.

**Ważne:** dokładny układ pól (Method/URL/Headers/Body) w surowym XML
akcji HTTP Request może się różnić między wersjami Taskera. Po imporcie
otwórz akcję `HTTP Request` w tasku „Messenger Links To Tasks” i
sprawdź/popraw wartości:

| Pole | Wartość |
|---|---|
| Method | `POST` |
| URL | `https://api.ticktick.com/open/v1/task` |
| Headers | `Content-Type: application/json`<br>`Authorization: Bearer %ticktick_token` |
| Body | `{"title":"Link od %evtprm2","content":"%links(1)"}` |

Jeśli chcesz, żeby zadanie trafiało do konkretnego projektu (listy) w
TickTick zamiast do Inbox, dodaj w Body pole `"projectId":"..."` z ID
projektu (znajdziesz je np. przez `GET /open/v1/project` w ich API).

## Test

1. Wyślij sobie testową wiadomość na Messengerze z linkiem od konta o
   nazwie wyświetlanej „Paulina Kuczkowska” (lub zmień warunek w akcji
   `If` na własne imię i nazwisko do testów).
2. Sprawdź, czy:
   - w pliku `messenger_links.txt` pojawiła się nowa linia,
   - w TickTick pojawiło się nowe zadanie z linkiem,
   - na telefonie pokazało się powiadomienie „Link od …”.
3. Wyślij link od kogoś innego — powinien trafić tylko do pliku i do
   powiadomienia, **bez** wpisu w TickTick.
