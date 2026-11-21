## 1. Cel projektu

Celem projektu jest stworzenie własnego bootloadera dla Raspberry Pi Pico, umożliwiającego aktualizowanie oprogramowania mikrokontrolera za pomocą interfejsu USB, bez konieczności ręcznego programowania pamięci Flash.

Aktualizacja będzie polegała na przesłaniu z komputera skompilowanego pliku `.bin`, który bootloader zapisze w pamięci Flash, zastępując dotychczasową aplikację użytkownika.

Projekt zostanie zrealizowany w języku C z wykorzystaniem Raspberry Pi Pico SDK.

## 2. Architektura rozwiązania

System będzie składał się z trzech elementów:

- **Bootloader (C)** – odpowiedzialny za rozruch mikrokontrolera, odbieranie firmware i zapisywanie go w pamięci Flash.
- **Biblioteka pomocnicza (C)** – udostępniająca aplikacji użytkownika funkcje obsługi USB i wykrywania żądania aktualizacji.
- **Program komputerowy (Python)** – odpowiedzialny za przesyłanie pliku `.bin` do mikrokontrolera.

Do komunikacji USB zostanie wykorzystany stos TinyUSB w trybie USB CDC (wirtualny port szeregowy). Nie będą wykorzystywane piny GPIO jako dodatkowy interfejs komunikacyjny.

## 3. Organizacja pamięci Flash

Pamięć Flash o pojemności 2 MB zostanie podzielona na trzy obszary:

| Obszar | Offset | Przeznaczenie |
|---|---|---|
| `boot2` | `0x000000–0x0000FF` | Standardowy drugi etap rozruchu Pico SDK |
| Bootloader | `0x000100–0x01FFFF` | Własny program rozruchowy |
| Aplikacja | `0x020000–0x1FFFFF` | Aktualizowany program użytkownika |

Stały adres bazowy aplikacji w przestrzeni XIP wyniesie `0x10020000`.

Dla bootloadera i aplikacji zostaną przygotowane osobne skrypty konsolidatora (`.ld`), zapewniające prawidłowe rozmieszczenie programów w pamięci.

Standardowy `boot2` pozostanie niezmieniony. Aktualizacja będzie obejmowała wyłącznie obszar pamięci przeznaczony na aplikację.

## 4. Mechanizm uruchamiania i inicjowania aktualizacji

Po każdym resecie własny bootloader sprawdzi specjalną flagę zapisaną w rejestrze Watchdog Scratch (`scratch[0]`).

- **Flaga nieustawiona:** nastąpi przekazanie sterowania do aplikacji użytkownika.
- **Flaga ustawiona:** bootloader wyzeruje flagę i przejdzie do trybu aktualizacji.

Przekazanie sterowania aplikacji będzie realizowane poprzez odczytanie jej tablicy wektorów, ustawienie rejestru VTOR, stosu oraz wykonanie skoku do procedury `Reset_Handler`.

Do wykonania końcowego skoku zostanie wykorzystany niewielki fragment asemblera ARM, natomiast pozostała część procedury zostanie napisana w C.

## 5. Biblioteka pomocnicza aplikacji

Aplikacja użytkownika będzie regularnie sprawdzać komunikację USB za pomocą biblioteki pomocniczej.

Biblioteka udostępni dwie podstawowe funkcje:

- `bootloader_init()` – inicjalizacja komunikacji USB.
- `bootloader_poll()` – obsługa USB i wykrywanie żądania aktualizacji.

Przykładowe wykorzystanie:

    int main(void)
    {
        bootloader_init();

        while (1)
        {
            bootloader_poll();

            application_task();
        }
    }

Po odebraniu specjalnej komendy `UPDATE` przez USB biblioteka zapisze znacznik w rejestrze Watchdog Scratch i wykona miękki reset mikrokontrolera.

Rozwiązanie będzie przeznaczone dla aplikacji działających w ciągłej pętli głównej, regularnie wywołujących `bootloader_poll()`.

## 6. Komunikacja komputer – mikrokontroler

Po stronie komputera zostanie przygotowany program w Pythonie z wykorzystaniem biblioteki `pySerial`.

Komunikacja będzie odbywała się przez USB CDC.

Przebieg aktualizacji:

1. Komputer nawiązuje połączenie z aplikacją i przesyła komendę `UPDATE`.
2. Aplikacja ustawia flagę i wykonuje miękki reset.
3. Bootloader wykrywa flagę i inicjalizuje TinyUSB.
4. Komputer ponownie nawiązuje połączenie z urządzeniem.
5. Bootloader przesyła komunikat `BOOTLOADER_READY`.
6. Komputer wysyła nagłówek `FWUP` zawierający rozmiar pliku `.bin` zapisany jako 4-bajtowa wartość `uint32_t`.
7. Po otrzymaniu potwierdzenia `READY` rozpoczyna przesyłanie danych.
8. Bootloader odbiera dane w blokach odpowiadających stronom Flash (256 B) i potwierdza ich zapis komunikatem `ACK`.
9. Po odebraniu całego pliku bootloader przesyła `DONE` i wykonuje reset.

Program komputerowy będzie odpowiedzialny za odczyt pliku, obsługę portu USB, przesyłanie danych oraz odbieranie potwierdzeń.

## 7. Operacje na pamięci Flash

Do obsługi pamięci zostaną wykorzystane funkcje Pico SDK:

- `flash_range_erase()` – kasowanie obszaru aplikacji.
- `flash_range_program()` – programowanie kolejnych stron pamięci.
- `flash_safe_execute()` – wykonywanie operacji z uwzględnieniem ograniczeń związanych z dostępem XIP.

Przyjęte parametry sprzętowe:

- sektor kasowania: 4096 B;
- strona programowania: 256 B.

Cały obszar aplikacji będzie kasowany jednym wywołaniem funkcji API, a następnie zapisywany kolejnymi stronami. Ostatnia niepełna strona zostanie uzupełniona bajtami `0xFF`.

Bootloader będzie kontrolował rozmiar przesyłanego pliku, aby uniemożliwić nadpisanie własnego obszaru pamięci.

## 8. Zakres projektu

Projekt obejmuje:
- własny bootloader;
- podział pamięci Flash i konfigurację linker scripts;
- procedurę skoku do aplikacji;
- niewielki fragment asemblera;
- obsługę flagi Watchdog i miękkiego resetu;
- bibliotekę pomocniczą aplikacji;
- komunikację USB CDC z wykorzystaniem TinyUSB;
- program komputerowy w Pythonie;
- przesyłanie pliku `.bin` i programowanie pamięci Flash.

Poza zakresem pozostają:
- aktualizacja samego bootloadera;
- przywracanie poprzedniej wersji aplikacji;
- alternatywne partycje aplikacji;
- szyfrowanie i cyfrowe podpisywanie firmware;
- CRC i mechanizmy ponawiania transmisji;
- odzyskiwanie po przerwaniu aktualizacji.

Zakłada się poprawny przebieg transmisji i operacji zapisu pamięci. Zachowana zostanie podstawowa kontrola rozmiaru firmware.

## 9. Oczekiwany rezultat

Rezultatem będzie działający system pozwalający na wielokrotne aktualizowanie aplikacji Raspberry Pi Pico przez USB, bez konieczności ręcznego korzystania z trybu BOOTSEL.

Demonstracja będzie polegała na uruchomieniu pierwszej wersji aplikacji, wysłaniu żądania aktualizacji przez USB, przesłaniu z komputera drugiej wersji programu i automatycznym uruchomieniu jej przez bootloader.
