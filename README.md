# Messenger Links → plik + TickTick (Tasker)

Profil Taskera, który nasłuchuje na powiadomienia Messengera, zapisuje
wykryte linki do pliku na telefonie, a jeśli wiadomość jest od wybranej
osoby (skonfigurowanej lokalnie, patrz niżej) — dodatkowo tworzy
przypomnienie w TickTick z tym linkiem.

## Ważne: dlaczego dwie akcje trzeba dodać ręcznie

Wcześniej próbowałem wpisać akcje „Write File” i „HTTP Request”
bezpośrednio jako surowy XML, zgadując ich numeryczne kody. To był błąd —
w Twojej wersji Taskera kod, którego użyłem dla „Write File” (129) to
faktycznie **JavaScriptlet**, a kod dla „HTTP Request” (339) w ogóle się
nie zaimportował. Numeracja kodów akcji różni się między wersjami
Taskera i nie da się jej bezpiecznie zgadnąć z zewnątrz — jedyny pewny
sposób to dodanie tych dwóch akcji ręcznie przez wyszukiwarkę akcji w
samym Tasker (wtedy aplikacja sama dobiera poprawny kod).

Plik `messenger_links.tasker.xml` zawiera więc tylko te fragmenty, które
są zweryfikowane (bo pochodzą 1:1 z Twojego oryginalnego, działającego
pliku, albo są prostymi warunkami `If`/`End If`), a w dwóch miejscach
zostawia **celowo pustą przestrzeń** do ręcznego uzupełnienia.

## Co jest w zaimportowanym szkielecie

Po imporcie zobaczysz w tasku „Messenger Links To Tasks v2” 4 kroki:

1. `If` — `%evtprm3 ~R (regex linku)` (dokładnie jak w oryginale)
2. `Variable Search Replace` — wyciąga link z `%evtprm3` do `%links` (jak w oryginale)
3. `If` — `%evtprm2 eq %target_contact` (nowe — filtr po osobie, patrz Krok 0)
4. `End If` (zamyka krok 3)

Na końcu jest jeszcze jeden `End If`, zamykający krok 1 — czyli w liście
zobaczysz razem 5 kroków.

**Musisz dodać ręcznie dwie akcje:**

### A. „Write File” — zaraz po kroku 2, przed krokiem 3

1. Otwórz task, dotknij `+` między krokiem 2 a 3.
2. Wybierz kategorię **File** → akcję **Write File**.
3. Ustaw:
   - **File**: `/storage/emulated/0/Documents/messenger_links.txt`
   - **Text**: `%TIMES %TIMEM | %evtprm2 | %links(1)`
   - **Add** (dopisywanie, nie nadpisywanie): włączone
   - **Add Newline**: włączone

Na Androidzie 11+ może być potrzebne przyznanie Taskerowi uprawnienia
„Zarządzanie wszystkimi plikami” w ustawieniach systemowych.

### B. „JavaScriptlet” (termin przypomnienia) + „HTTP Request” do TickTick — wewnątrz drugiego `If` (między krokiem 3 a 4)

TickTick wymaga, żeby termin (`dueDate`) był w formacie ISO 8601 UTC —
prościej i pewniej wyliczyć go jedną akcją JavaScriptlet niż zgadywać
składnię matematyki dat w zwykłym polu tekstowym Taskera.

1. Dotknij `+` zaraz po kroku 3 (czyli wewnątrz `If %evtprm2 eq %target_contact`).
2. Wyszukaj akcję **JavaScriptlet** i wklej kod:
   ```javascript
   var d = new Date(Date.now() + 2*60*60*1000);
   setGlobal("due_date", d.toISOString());
   ```
   (`2*60*60*1000` = 2 godziny w milisekundach — zmień na inną wartość,
   jeśli chcesz inny odstęp).
3. Zaraz po tej akcji dodaj kolejną: wyszukaj **HTTP Request**
   (kategoria Net; jeśli Twoja wersja Taskera jej nie ma, poszukaj
   **HTTP Post**).
4. Ustaw:

   | Pole | Wartość |
   |---|---|
   | Method | `POST` |
   | URL | `https://api.ticktick.com/open/v1/task` |
   | Headers | `Content-Type: application/json`<br>`Authorization: Bearer %ticktick_token` |
   | Body | `{"title":"Link od %evtprm2","content":"%links(1)","priority":3,"dueDate":"%due_date","reminders":["TRIGGER:PT0S"],"timeZone":"Europe/Warsaw"}` |

Co robią dodatkowe pola w Body:
- `"priority":3` — priorytet Średni (0=brak, 1=Niski, 3=Średni, 5=Wysoki).
- `"dueDate":"%due_date"` — termin obliczony w kroku 2.
- `"reminders":["TRIGGER:PT0S"]` — przypomnienie dokładnie w momencie terminu.
- `"timeZone":"Europe/Warsaw"` — strefa czasowa do poprawnej interpretacji terminu.

Jeśli chcesz, żeby zadanie trafiało do konkretnego projektu (listy) w
TickTick zamiast do Inbox, dodaj w Body pole `"projectId":"..."` z ID
projektu (znajdziesz je np. przez `GET /open/v1/project` w ich API).

Po dodaniu wszystkich akcji cała lista kroków w tasku powinna wyglądać tak:

1. `If` (regex linku)
2. `Variable Search Replace`
3. `Write File` ← dodane ręcznie
4. `If %evtprm2 eq %target_contact`
5. `JavaScriptlet` (oblicza `%due_date`) ← dodane ręcznie
6. `HTTP Request` (TickTick) ← dodane ręcznie
7. `End If`
8. `End If`

## Import do Taskera

1. Skopiuj plik `messenger_links.tasker.xml` na telefon (np. przez Google
   Drive, e-mail do siebie, albo `adb push`).
2. **Usuń każdą wcześniejszą wersję** tego profilu i tasku przed
   ponownym importem: w zakładce **Tasks** znajdź i usuń wszystkie taski
   „Messenger Links To Tasks v2” (może być więcej niż jeden), potem w
   zakładce **Profiles** usuń profil „Messenger Links v2”. Import z tym
   samym ID potrafi doklejać stare akcje do nowych zamiast je czysto
   nadpisać.
3. W Taskerze: **Profile → długie przytrzymanie w pustym miejscu → Import
   Profile** i wskaż plik.
4. Zaakceptuj uprawnienia, o które poprosi Tasker (dostęp do powiadomień,
   dostęp do pamięci).
5. Dodaj ręcznie akcje A i B opisane wyżej.

Po imporcie sprawdź też:
- czy profil „Messenger Links v2” jest **włączony** (pasek u góry profilu w
  Tasker powinien być kolorowy, nie wyszarzony),
- czy dostęp do powiadomień dla Taskera jest wciąż aktywny w Ustawienia →
  Aplikacje → Dostęp specjalny → Dostęp do powiadomień.

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

## Krok 1: token API TickTick (wymagane ręcznie)

1. Wejdź na https://developer.ticktick.com/manage i zarejestruj
   aplikację (Client ID/Secret) — potrzebne do przeprowadzenia
   autoryzacji OAuth2.
2. Przeprowadź przepływ OAuth2 (authorization code), żeby uzyskać
   **access token** (patrz historia tej rozmowy/repo dla dokładnych
   kroków z `curl`/PowerShell).
3. W Taskerze utwórz zmienną globalną `%ticktick_token` z wartością
   tokena: dowolny task jednorazowy → akcja `Variable Set` → uruchom raz
   ręcznie.

Token ma zwykle ~180 dni ważności (`expires_in` w odpowiedzi) — po tym
czasie trzeba będzie powtórzyć autoryzację.

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
  nie próbuj ich używać w warunkach/akcjach.

## Wiadomości z podglądem linku (zdjęcie) — wciąż do zdiagnozowania

Messenger dla wiadomości z linkiem, który ma podgląd (miniaturkę), może
pokazywać krótkie powiadomienie typu „Wysłała zdjęcie” zamiast samego
linku w `%evtprm3`. Skoro `%evtprm4`/`%evtprm5` nie istnieją, nie wiemy
jeszcze, w którym dokładnie polu (jeśli w ogóle w jakimś oddzielnym) ląduje
link dla takiej wiadomości — trzeba to sprawdzić profilem DIAG opisanym
wyżej, wysyłając sobie realną wiadomość z podglądem linku (nie zwykły
tekst) i patrząc, co faktycznie pokaże się w `%evtprm1`-`%evtprm5`. Jeśli
link nigdzie się nie pojawi, może się okazać, że trzeba złapać zupełnie
inne pole notification, którego obecne Tasker Notification event może
nie eksponować wprost — wtedy zgłoś mi dokładną treść z DIAG.

## Test

1. Wyślij sobie testową wiadomość na Messengerze z linkiem od konta o
   nazwie wyświetlanej takiej samej jak wartość `%target_contact`
   ustawiona w Kroku 0.
2. Sprawdź, czy:
   - w pliku `messenger_links.txt` pojawiła się nowa linia,
   - w TickTick pojawiło się nowe zadanie z linkiem.
3. Wyślij link od kogoś innego — powinien trafić tylko do pliku,
   **bez** wpisu w TickTick.
