# Redragon K686-RGB-PRO – protokół konfiguracji (odbiornik 2.4G 3554:fa09)

Źródło: analiza statyczna `OemDrv.exe` (MFC, x86, z instalatora Inno Setup 5.3.3
`Redragon_K686-RGB-PRO_Setup_v2.0_20240110`, SHA-256 `b7e01ed0…386b1`) oraz plików
`app/Dev/kb/KB.ini`, `app/Text/en/text.xml`, a także deskryptorów HID odczytanych z `/sys`.
Adresy w nawiasach to adresy funkcji w `OemDrv.exe` (ImageBase 0x400000).

Oznaczenia: **[PEWNE]** = wynika wprost z kodu/deskryptora, **[HIPOTEZA]** = wniosek, wymaga potwierdzenia na urządzeniu.

## 1. Co robi aplikacja

Aplikacja to generyczny sterownik „BYCOMBO4” obsługujący kilka rodzin urządzeń (klasy `CDev3632`,
`CDev916KB`, `CDevG5KB`, `CDevG5MS`, `CDevComboFilm`). Dla tej klawiatury `KB.ini` definiuje:

| Funkcja | Źródło |
|---|---|
| Efekty podświetlenia (20 pozycji, w tym Self-define i OFF), jasność 0–4, prędkość 0–4, kolor z palety 7 kolorów lub „Colorful” | `LedOpt1..20`, `Light/LightHW`, `Speed/SpeedHW`, `text.xml` |
| Paleta kolorów (7 kolorów na efekt) | komenda 0x09/0x49 |
| Kolory per-klawisz (tryb Self-define, `GaoshouGroupNum=1`) | komenda 0x02/0x42 |
| Remapowanie klawiszy, 4 warstwy (`LayerNum=4`), Fn | komenda 0x01/0x41 |
| Makra (bufor 1024 B, `MacroBufferSize=1024`) | komenda 0x03/0x43 |
| Usypianie w trybie bezprzewodowym (`SleepTime=180`) | pole w bloku LED [HIPOTEZA] |
| Aktualizacja firmware | **tylko przez kabel USB** („Please switch the device to USB connection first”) – nie przez odbiornik |
| Profil w chmurze (logowanie, upload makr) | HTTP do 120.79.152.79 – pominięte |

Komunikacja: **[PEWNE]** wyłącznie HID przez `hid.dll`/`setupapi.dll`: dla odbiornika `WriteFile`/`ReadFile`
(raporty Output/Input), a dla trybu przewodowego `HidD_SetFeature`/`HidD_GetFeature`.
Wersja firmware odbiornika to `bcdDevice` z `HidD_GetAttributes` (u Ciebie 0x0102), bez osobnej komendy.

## 2. Identyfikacja urządzenia

| | Wartość | Status |
|---|---|---|
| Odbiornik 2.4G | VID 0x3554, PID 0xFA09 („CX 2.4G Wireless Receiver”) | [PEWNE] |
| Przewodowo | VID 0x258A, PID 0x010C | [PEWNE] z KB.ini (nie podłączano) |
| Interfejs konfiguracyjny | **interfejs 1** (`hidraw4`), kolekcja UsagePage **0xFF02**, Usage **0x02** | [PEWNE] |
| Report ID | **0x13**, Input 19 B + Output 19 B (20 B z ID) | [PEWNE] |
| Wybór klasy w aplikacji | `UsagePage==0xFF02 && Usage==2 && InLen==20 && OutLen==20 && RID==0x13` ⇒ `CDev3632` (0x40c140) | [PEWNE] |
| Interfejs 0 | klawiatura boot, bez kanału konfiguracji | [PEWNE] |
| Report 0x06 (Feature, 7 B, strona 0xFF04) | aplikacja go nie używa dla tego urządzenia | [PEWNE] dla `CDev3632` |

Interfejs 1 ma tylko endpoint IN (0x82), więc raporty Output idą przez SET_REPORT (control).
WebHID i Linux obsługują to automatycznie.

## 3. Format ramki (Report 0x13, 20 bajtów z ID)

Indeksy `b[]` liczone **z Report ID**. W WebHID `sendReport(0x13, data)` gdzie `data = b[1..19]`.

```
b[0]  = 0x13                      Report ID
b[1]  = cmd                       (odpowiedź: bit7=1 → błąd CRC zgłoszony przez urządzenie)
b[2]  = liczba pakietów (total)   (w ACK: bit7=1 → błąd CRC)
b[3]  = indeks pakietu (0..total-1)
b[4]  = (sub << 4) | len          sub = warstwa/blok/grupa, len = liczba bajtów danych (0..14)
b[5..18] = dane (max 14 B)
b[19] = suma kontrolna = (b[0] + b[1] + … + b[18]) & 0xFF   (Report ID wliczony!)
```
[PEWNE] Funkcja sumy to 0x485d10, a budowa pakietów jest w 0x485440 (zapis) i 0x4856e0/0x486000 (odczyt).

### Odczyt (Get*)
Host wysyła 1 ramkę: `13 cmd 01 00 (sub<<4) 00…00 sum`. Urządzenie odpowiada pakietami z
`b[1]==cmd`, `b[2]=total`, `b[3]=idx`, `b[4]&0x0F=len`, a dane są w `b[5..]`. Odczyt kończy się, gdy `idx+1==total`.
Timeout to 1000 ms na pakiet, aplikacja ponawia do 5 razy.

### Zapis (Set*)
Dane są dzielone na pakiety po 14 B. Każdy pakiet: `13 cmd total idx (sub<<4|len) dane… sum`.
Przed każdym pakietem jest opóźnienie 15 ms. Po każdym pakiecie trzeba czekać na ACK (ramka z `b[1]==cmd`)
do 1000 ms. Przy timeoucie albo bicie CRC aplikacja ponawia ten sam pakiet do 5 razy.
Komendy z bitem 7 (0x88, 0x8E) wysyłane są bez ACK.

### Zdarzenia asynchroniczne
`b[1]==0x0A` to powiadomienie: `b[5]`=typ, `b[6]`=wartość. Typ 2 to stan połączenia klawiatura↔odbiornik
(1 = połączona). [PEWNE] z 0x4850d0. Pozostałe typy obsługuje `OnDeviceBehavior` (0x457680):
typ 3 = poziom DPI (myszy), typ 7 = zmieniono ustawienia LED, z Fn lub po zapisie z hosta (aplikacja ignoruje; obserwacja z urządzenia), typ 5 = **poziom baterii** (zapisywany w stanie urządzenia, pokazywany przy `ShowPower`).
Przykład z urządzenia: `13 0a 01 00 04 05 64 01 00 … 8c`, czyli bateria 100% (`0x64`), kolejny bajt `01` ma nieznane znaczenie.
Zdarzenia mają tę samą ramkę co odpowiedzi (b[4] = długość).

Pierwsze żądanie po dłuższej przerwie często kończy się timeoutem (uśpione radio); ponowienie działa.
Aplikacja ponawia do 5 razy.

## 4. Komendy (klasa CDev3632, vtable 0x60c8ec)

| cmd | Kierunek | sub | Dane | Funkcja | Status |
|---|---|---|---|---|---|
| 0x05 | odczyt (odbiornik) | 0 | 10 B (aplikacja bierze 6) | PSD, czyli identyfikator produktu; KB.ini `Psd=3,0,0,0,1,20` jest parsowane **szesnastkowo**, więc oczekiwane `03 00 00 00 01 20` | [POTWIERDZONE na urządzeniu]: `03 00 00 00 01 20 00 00 10 00` |
| 0x07 | odczyt (odbiornik) | 0 | 1 B | Online: czy klawiatura jest połączona z odbiornikiem | [POTWIERDZONE]: `01` |
| 0x44 | odczyt | 0 | 128 B | GetLED: blok konfiguracji oświetlenia | [PEWNE] |
| 0x04 | zapis | 0 | 128 B | SetLED | [PEWNE] |
| 0x49 | odczyt | 0 | ≤0x200 B | GetLedRgbTab: palety kolorów | [PEWNE] |
| 0x09 | zapis | 0 | 0x200 B (CRC=1) | SetLedRgbTab | [PEWNE] |
| 0x4A | odczyt | 0 | 2 B | W aplikacji `GetOnBoard` (1 B), ale K686 zwraca `64 01`, te same wartości co zdarzenie baterii (typ 5) | [POTWIERDZONE] odpowiedź, [HIPOTEZA] bateria % + flaga ładowania |
| 0x0A | zapis | 0 | 1 B | SetOnBoard (uwaga: kolizja kodu ze zdarzeniami 0x0A) | [PEWNE], nie wdrożone |
| 0x41 | odczyt | warstwa 0–3 | 512 B (128×4) | GetMatrix: mapa klawiszy | [PEWNE] |
| 0x01 | zapis | warstwa 0–3 | 512 B | SetMatrix: remap | [PEWNE], nie wdrożone |
| 0x43 | odczyt | 0 | ≤1024 B | GetMacro | [PEWNE] |
| 0x03 | zapis | blok 512 B | po 0x200 B | SetMacro | [PEWNE], nie wdrożone |
| 0x42 | odczyt | grupa | 384 B | GetGame: kolory per-klawisz | [PEWNE] |
| 0x02 | zapis | grupa | 3×128 B | SetGame/SetLedMatrix (Self-define) | [PEWNE], nie wdrożone |
| 0x0B | zapis | 0 | ? | SetScreenParam (nie dotyczy K686) | pominięte |
| 0x88 | zapis bez ACK | 0 | ? | SetRealData (podgląd na żywo) | pominięte |
| 0x8E / 0x80+x | zapis bez ACK | ? | ? | tryb muzyczny | pominięte |

Komend firmware/bootloadera **nie ma** w klasie `CDev3632`: aktualizacja odbywa się tylko po kablu.

## 5. Blok LED (128 B, cmd 0x44/0x04)

Ze składania bloku w 0x48c760 (aplikacja zawsze czyta 0x44, modyfikuje i zapisuje 0x04):

| Offset | Znaczenie | Status |
|---|---|---|
| 0x09 | 1 = tryb Self-define (`GaoshouIndex=18`, tj. pozycja 19), inaczej 0 | [PEWNE] logika, [HIPOTEZA] znaczenie |
| 0x0A | sprzętowy kod efektu = kolumna `hw` z `LedOpt` | [PEWNE] |
| 0x0B | `StartGp` (domyślnie 0x20) + numer grupy kolorów | [PEWNE] logika |
| 0x16 | czas uśpienia (0 = wyłączone) | [HIPOTEZA] |
| 0x38, 0x39 | aplikacja zawsze wpisuje 0xFF | [PEWNE] |
| 0x3A + 2·i | jasność efektu i (0–4, `LightHW`) | [PEWNE] logika, [HIPOTEZA] indeksowanie |
| 0x3B + 2·i | `(prędkość << 4) \| kolor`: prędkość 0–4 (`SpeedHW`), kolor 0–6 = indeks w palecie, 7 = Colorful (`RandomVal`) | jw. |

`i` = pozycja efektu w `LedOpt` (0..19), czyli 20 slotów pod offsetami 0x3A..0x61.

**Weryfikacja na urządzeniu (2026-10-25):**
- Odczyt: `[0x0A]=0x0C` (hw 12 = Reaction), `[0x09]=0`, `[0x0B]=0x20` (`StartGp`), `[0x38..0x39]=FF FF`.
- Sloty mają postać `04 47`, czyli jasność 4, prędkość 4, kolor 7 (Colorful). Slot 11 (Reaction) to `04 40` (kolor 0).
- Po obniżeniu jasności klawiszem Fn zmienił się **tylko bajt 0x50** = 0x3A+2·11 (04 → 00). Potwierdza to indeksowanie slotów i jasność 0–4. [POTWIERDZONE]
- Bajty 0x62..0x7D (`07 47 … 07 44 … 04 04 … 04 14 …`) mają nieznane znaczenie. Strona ich nie zmienia.
- **Ostatnie 2 bajty bloku (0x7E–0x7F) to `5A A5`**, czyli znacznik końca przechowywany w urządzeniu. Przy odczycie-modyfikacji-zapisie zostaje zachowany.
- Podczas zmiany jasności klawiszem Fn klawiatura wysyła zdarzenia `0A … 04 07 00 …` (typ 7).
- **Zapis potwierdzony (2026-10-25 14:54):** SetLED (0x04) z `[0x50] 04 → 02`. Wysłano 10 pakietów co ~30 ms.
  ACK to **dokładne echo wysłanej ramki** (ta sama suma). Odczyt po zapisie był zgodny.
  Około 0,5 s po zapisie przyszło zdarzenie typu 7, więc typ 7 = „zmieniono ustawienia LED” (z Fn lub z hosta).

### Efekty (KB.ini `LedOpt`, kolumny: hw, anim, speed, light, direct, random, color)

| i | Nazwa (`tc_kb_led{i+1}`) | hw | prędkość | jasność | Colorful | kolor |
|---|---|---|---|---|---|---|
| 0 | Fixed_on | 1 | – | ✓ | ✓ | ✓ |
| 1 | Respire | 2 | ✓ | ✓ | ✓ | ✓ |
| 2 | Rainbow | 3 | ✓ | ✓ | – | – |
| 3 | Flash_away | 4 | ✓ | ✓ | ✓ | ✓ |
| 4 | Raindrops | 5 | ✓ | ✓ | ✓ | ✓ |
| 5 | Rainbow_wheel | 6 | ✓ | ✓ | ✓ | ✓ |
| 6 | Ripples_shining | 7 | ✓ | ✓ | ✓ | ✓ |
| 7 | Stars_twinkle | 8 | ✓ | ✓ | ✓ | ✓ |
| 8 | Shadow_disappear | 9 | ✓ | ✓ | ✓ | ✓ |
| 9 | Retro_snake | 10 | ✓ | ✓ | ✓ | ✓ |
| 10 | Neon_stream | 11 | ✓ | ✓ | ✓ | ✓ |
| 11 | Reaction | 12 | ✓ | ✓ | ✓ | ✓ |
| 12 | Sine_wave | 13 | ✓ | ✓ | ✓ | ✓ |
| 13 | Retinue scanning | 14 | ✓ | ✓ | ✓ | ✓ |
| 14 | Rotating windmill | 15 | ✓ | ✓ | – | – |
| 15 | Colorful waterfall | 16 | ✓ | ✓ | – | – |
| 16 | Blossoming | 17 | ✓ | ✓ | – | – |
| 17 | Rotating storm | 18 | ✓ | ✓ | ✓ | ✓ |
| 18 | Self-define | 21 | – | – | – | – |
| 19 | OFF | 0 | – | – | – | – |

Przypisanie nazwy do pozycji (a nie do kolumny „anim”) to [HIPOTEZA] o wysokim prawdopodobieństwie:
efekty bez wyboru koloru to dokładnie Rainbow/Windmill/Waterfall/Blossoming, a Fixed_on nie ma prędkości.

## 6. Paleta kolorów (cmd 0x49/0x09)

Wpis `k` = i+1 (wpis 0 nieużywany) zajmuje 21 B pod offsetem `21·k`, czyli 7 kolorów × 3 B.
Kolejność bajtów wynika z `RGBIndex=0,1,2`, więc jest to **R, G, B**.
Przy `CRC=1` (tak jest w KB.ini) bufor ma 0x200 B, a pod offsetami 0x1FA/0x1FB są bajty `5A A5`. [PEWNE] z 0x48d2a1.

**Weryfikacja na urządzeniu:**
- Odczyt 0x49 zwraca **490 B (35 pakietów)**, bez znacznika `5A A5`.
- Wpisy 1–13 zawierają domyślną paletę: czerwony, zielony, niebieski, żółty, magenta, cyjan, biały. To potwierdza kolejność **R,G,B**. [POTWIERDZONE]
- Wpis 12 (Reaction) ma kolor 0 = `7F C4 FE`, co zgadza się z kolorem 0 w slocie Reaction.
- Wpisy 14–20 i dalsze bajty to zera.
- Strona zapisuje paletę tak jak aplikacja: 0x200 B = 490 B z odczytu + zera + `5A A5` pod 0x1FA.
- Przy pierwszym odczycie zgubiły się 3 pakiety (seria co ~4 ms), ponowienie zadziałało.

## 7. Kolory per-klawisz (cmd 0x42/0x02)

Trzy płaszczyzny po 0x7E (126) B: R[idx], G[126+idx], B[252+idx]. Wysyłane 3·`MatrixLen` = 384 B.
`idx` = 8. kolumna sekcji `[KEY]` w KB.ini (np. Esc=0, F1=6, `=1). [PEWNE] logika, [HIPOTEZA] że idx = numer LED.

## 8. Mapa klawiszy (cmd 0x41/0x01)

128 pozycji × 4 B na warstwę, indeks = `idx` z `[KEY]`. Kodowanie wartości (funkcja
`keyinfo_to_hardware_code`) nie zostało jeszcze zanalizowane, więc remap i makra pozostają tylko do odczytu.

## 9. Przechwyty USB – kiedy będą potrzebne

Na odczyty i zmiany LED **nie są potrzebne**. Przydadzą się do:
1. dekodowania wartości mapy klawiszy i formatu makr (remap/makra),
2. potwierdzenia hipotez z sekcji 5–7, jeśli odczyt z urządzenia im przeczy,
3. ewentualnie trybu przewodowego 258a:010c (inny protokół: Feature reports, klasa `CDev916KB`).
