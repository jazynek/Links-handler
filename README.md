# Messenger Links → plik + TickTick (Tasker)

Profil Taskera, który nasłuchuje na powiadomienia Messengera, zapisuje
wykryte linki do pliku na telefonie, a jeśli wiadomość jest od wybranej
osoby (skonfigurowanej lokalnie, patrz niżej) — dodatkowo tworzy
przypomnienie w TickTick z tym linkiem.

## Co zmieniło się względem oryginału

- **Profil** (`Messenger Links`) już nie jest ograniczony samym tytułem
  powiadomienia do jednej osoby — teraz reaguje na powiadomienia
  z Messengera od **dowolnej osoby**, ale nadal tylko wtedy, gdy treść
  zawiera link (warunek regex w akcji If bez zmian).
- **Task** (`Messenger Links To Tasks`) ma nowe akcje:
  1. `If` (regex linku) — bez zmian.
  2. `Matches Regex` → wyciąga link do `%links` — bez zmian.
  3. **`Write File`** (nowe) — dopisuje linię z datą, nadawcą i linkiem do
     pliku `messenger_links.txt`.
  4. **`If %evtprm2 = %target_contact`** (nowe) — dalsze kroki tylko dla
     osoby wskazanej w zmiennej `%target_contact` (patrz "Krok 0" niżej).
  5. **`HTTP Request`** (nowe) — POST do TickTick Open API, tworzy zadanie
     z linkiem w treści.
  6. `End If`
  7. `Notify` — powiadomienie na telefonie, jak wcześniej (teraz dla
     każdego nadawcy).
  8. `End If`

## Import do Taskera

1. Skopiuj plik `messenger_links.tasker.xml` na telefon (np. przez Google
   Drive, e-mail do siebie, albo `adb push`).
2. W Taskerze: **Profile → długie przytrzymanie w pustym miejscu → Import
   Profile** i wskaż plik (albo otwórz plik menedżerem plików — Tasker
   powinien zaproponować import).
3. Zaakceptuj uprawnienia, o które poprosi Tasker (dostęp do powiadomień,
   dostęp do pamięci).

**Jeśli wcześniej importowałeś starszą wersję tego profilu**, przed
ponownym importem usuń starą wersję (długie przytrzymanie na profilu
„Messenger Links” → Delete). Import z tym samym ID czasem tworzy duplikat
albo nie nadpisuje wszystkich pól poprawnie, co może wyglądać jak "pusty"
trigger.

Po imporcie sprawdź też:
- czy profil „Messenger Links” jest **włączony** (pasek u góry profilu w
  Tasker powinien być kolorowy, nie wyszarzony),
- czy dostęp do powiadomień dla Taskera jest wciąż aktywny w Ustawienia →
  Aplikacje → Dostęp specjalny → Dostęp do powiadomień.

**Uwaga o polu filtru tytułu** w zdarzeniu Notification (`arg1`): w
oryginalnym pliku miało ono zwykły tekst („Paulina Kuczkowska”), bez
składni regex — czyli robi zwykłe dopasowanie tekstu, nie regex. Zostawiam
je teraz **puste**, co w Tasker oznacza „brak filtra, dopasuj każdy
tytuł” (dokładnie tak samo puste są pola `arg2`-`arg6` w oryginale i to
nigdy nie było problemem). Jeśli mimo to profil nadal się nie uruchamia,
to prawdopodobnie w Twojej wersji Taskera puste pole rzeczywiście nie
działa jako wildcard — w takim razie zamiast zostawiać pole puste, wpisz
w nim `%target_contact` (tę samą zmienną, którą i tak ustawiasz w Kroku
0), co odtworzy dokładnie sprawdzone, oryginalne zachowanie (filtr po
konkretnej osobie), tylko bez wpisanego na sztywno imienia i nazwiska w
repozytorium. W tym wariancie zapis do pliku i powiadomienie też będą
tylko dla tej jednej osoby — czyli wracamy do zakresu z oryginalnego
pliku, zamiast łapania linków od wszystkich.

## Krok 0: kogo pilnujemy (TickTick)

Imię i nazwisko osoby, dla której mają powstawać przypomnienia w
TickTick, nie jest wpisane w tym repozytorium — jest ustawiane lokalnie,
w Tasker, jako zmienna globalna `%target_contact`.

**Ważne:** `%evtprm2` to nie zawsze pełne imię i nazwisko — może to być
pseudonim/nazwa, jaką masz zapisaną w kontaktach albo w czacie
Messengera (np. w testach na jednym telefonie pojawiła się wartość
„Kucza” zamiast pełnego imienia i nazwiska). Zanim ustawisz
`%target_contact`, sprawdź dokładną wartość `%evtprm2` dla realnej
wiadomości od tej osoby — najprościej profilem diagnostycznym opisanym w
sekcji „Diagnostyka” niżej.

W Tasker: dowolny task jednorazowy → akcja `Variable Set`, Name =
`%target_contact`, To = dokładna wartość, jaką faktycznie zobaczysz w
`%evtprm2` (znak w znak, łącznie z wielkością liter). Uruchom raz
ręcznie.

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

## Diagnostyka: co naprawdę jest w %evtprm

W tym repo jest osobny plik **`diagnostyka_powiadomien.tasker.xml`** —
minimalny profil bez żadnych filtrów/warunków, który przy **każdym**
powiadomieniu na telefonie pokazuje treść `%evtprm1`-`%evtprm5`.
Zaimportuj go (to osobny profil „DIAG Notification Test”, nie koliduje z
„Messenger Links”), wywołaj powiadomienie które chcesz zdiagnozować, a
potem usuń profil DIAG, gdy skończysz.

Co już z niego wiemy (na podstawie testu na jednym telefonie):
- `%evtprm2` = nazwa/tytuł z powiadomienia (może to być pseudonim, nie
  pełne imię i nazwisko — patrz „Krok 0” wyżej).
- `%evtprm3` = treść powiadomienia.
- `%evtprm4` i `%evtprm5` **nie istnieją** dla tego zdarzenia w Tasker —
  próba użycia ich (`%evtprm3%evtprm4`) dokładała do treści dosłowny,
  niepodstawiony tekst `%evtprm4`, co psuło dopasowanie regexu (warunek
  w akcji `If` robi pełne dopasowanie do całego tekstu, więc doklejony
  „śmieć” na końcu unieważniał nawet zwykłe, wcześniej działające
  linki). **Dlatego cofnąłem warunek i wyciąganie linku z powrotem do
  samego `%evtprm3`.**

## Wiadomości z podglądem linku (zdjęcie) — wciąż do zdiagnozowania

Messenger dla wiadomości z linkiem, który ma podgląd (miniaturkę), może
pokazywać krótkie powiadomienie typu „Wysłała zdjęcie” zamiast samego
linku w `%evtprm3`. Skoro `%evtprm4`/`%evtprm5` nie istnieją, nie wiemy
jeszcze, w którym dokładnie polu (jeśli w ogóle w jakimś oddzielnym) ląduje
link dla takiej wiadomości — trzeba to sprawdzić profilem DIAG opisanym
wyżej, wysyłając sobie realną wiadomość z podglądem linku (nie zwykły
tekst) i patrząc, co faktycznie pokaże się w `%evtprm1`-`%evtprm5`. Jeśli
link nigdzie się nie pojawi, może się okazać, że trzeba złapać zupełnie
inne pole notification (np. `actions`/`extras`), którego obecne Tasker
Notification event może nie eksponować wprost — wtedy zgłoś mi dokładną
treść z DIAG, dopasujemy rozwiązanie do realnych danych.

## Test

1. Wyślij sobie testową wiadomość na Messengerze z linkiem od konta o
   nazwie wyświetlanej takiej samej jak wartość `%target_contact`
   ustawiona w Kroku 0.
2. Sprawdź, czy:
   - w pliku `messenger_links.txt` pojawiła się nowa linia,
   - w TickTick pojawiło się nowe zadanie z linkiem,
   - na telefonie pokazało się powiadomienie „Link od …”.
3. Wyślij link od kogoś innego — powinien trafić tylko do pliku i do
   powiadomienia, **bez** wpisu w TickTick.
