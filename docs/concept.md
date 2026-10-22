# Dokumentacja Koncepcyjna Projektu

**Temat pracy:** *Implementacja bezpośredniego interfejsu sensor-mikrokontroler dla czujników rezystancyjnych.*

## 1. Cel i Założenia Projektu

- **Główny cel:** Zbudowanie autorskiego stanowiska pomiarowego do przeprowadzenia metrologicznego porównania trzech algorytmów kalibracji układów DIC (1-, 2- i 3-punktowej).
- **Cel porównawczy:** Zbadanie wpływu architektury mikrokontrolera (8-bit AVR vs 32-bit ARM Cortex-M) oraz częstotliwości zegara referencyjnego ($T_{ref}$) na rozdzielczość TDC (Time-to-Digital Conversion), szum wyzwalania i błąd kwantyzacji.
- **Rzetelność badań:** Obydwa mikrokontrolery mierzą **ten sam pojedynczy tor pomiarowy**, a mikrokontroler nieaktywny jest galwanicznie odcinany kluczem analogowym i trzymany w stanie sprzętowego RESETu (stan *High-Z* na pinach I/O).

- **Rzetelność badań:** Obydwa mikrokontrolery mierzą **ten sam pojedynczy tor pomiarowy**, a mikrokontroler nieaktywny jest galwanicznie odcinany kluczem analogowym i trzymany w stanie sprzętowego RESETu (stan *High-Z* na pinach I/O).

## 2. Metodologia i Algorytmy Kalibracji (RC-DIC)

Pomiar w układzie DIC polega na wyznaczeniu czasu rozładowania kondensatora $C$ przez badany rezystor do dolnego progu bufora Schmitta $V_{TL}$. Czas $T_d$ przeliczany jest na cykle timera $N_x$.

| Metoda Kalibracji | Wymagane Elementy Wzorcowe | Wzór Przeliczeniowy | Eliminowane Błędy i Zalety |
| :--- | :--- | :--- | :--- |
| **Jednopunktowa** | 1 rezystor wzorcowy $R_{c2}$ | $R_x = \frac{N_x}{N_{c2}} R_{c2}$ | Eliminuje zmiany pojemności $C$, napięcia $V_{cc}$, zegara $T_{ref}$ i progu $V_{TL}$. Zakłada brak offsetu. |
| **Dwupunktowa** | 2 rezystory wzorcowe $R_{c1}, R_{c2}$ | $R_x = R_{c1} + \frac{N_x - N_{c1}}{N_{c2} - N_{c1}}(R_{c2} - R_{c1})$ | Kompensuje stały błąd zerowy (offset), opóźnienia i liniową część rezystancji pinów $R_{OL}$. |
| **Trzypunktowa** | Rezystor $R_{c2}$ + Zwarcie + Rezystor prądowy $R_0$ | $R_x = \frac{N_x - N_{c1}}{N_{c2} - N_{c1}} R_{c2}$ | **Eliminuje wszystkie powyższe** + rezystancję wyjściową tranzystorów GPIO ($R_{OL}$) oraz opóźnienie programowe. Błąd $< 0.01\%$. |

> **Wniosek:** Układ do kalibracji 3-punktowej jest nadzbiorem pozostałych metod. Przełączając piny w kodzie programowym, na jednym fizycznym torze pomiarowym realizowane są wszystkie trzy algorytmy.

## 3. Architektura Sprzętowa

### 3.1. Diagram blokowy

```mermaid
  flowchart TB
    subgraph InterfaceSection["Mostek USB, Zasilanie i Układ Reset"]
        LDO["LDO AP7361C\n(5V USB ➔ 3.3V)"]:::pwrStyle
        CP2105["Mostek Dual-UART CP2105\n(VIO zasilane z 3.3V LDO)"]:::commStyle
        RST_STM["Reset STM32\n(Dioda Schottky BAT54 + Pull-up 10k)"]:::rstStyle
        RST_AVR["Reset AVR\n(Dioda Schottky BAT54 + Pull-up 10k)"]:::rstStyle
        MUX_JUMP{"Wybór MUX_SEL\n(Zworka: RTS vs GPIO)"}:::commStyle
    end

    subgraph MCUSection["Badane MCU"]
        STM32["STM32L073RZ\n• Taktowanie: 32 MHz (Tref = 31.25 ns)\n• Wejście pomiarowe: TIM_CH1 (Input Capture)\n• Tryb pomiarowy: Sleep Mode"]:::mcuStyle
        AVR["ATmega328P\n• Taktowanie: 8 MHz Kwarc (Tref = 125 ns)\n• Pomiar: wejście ICP1 (input capture)\n• Tryb pomiarowy: Idle Mode"]:::mcuStyle
    end

    MUX["Klucz Analogowy 6-kanałowy 1:2 (TS3A27518E)\n• Ron ≈ 4.4 Ω, Izolacja linii I/O nieaktywnego MCU"]:::muxStyle

    subgraph CircuitSection["Wspólny Tor Pomiarowy"]
        direction TB
        RX["Rezystor Badany Rx\n(Czujnik rezystancyjny)"]:::circuitStyle
        RC1["Wzorzec Rc1\n(Dolny: 200 Ω, 0.1%)"]:::circuitStyle
        RC2["Wzorzec Rc2\n(Górny: 10 kΩ, 0.1%)"]:::circuitStyle
        SHORT["Ścieżka zwarciowa\n(R = 0 Ω)"]:::circuitStyle

        SUM_NODE["Wspólny węzeł magistrali"]:::nodeStyle
        R0["Rezystor szeregowy R0\n(100 Ω, 0.1%)"]:::circuitStyle
        VO_NODE["Węzeł pomiarowy Vo"]:::nodeStyle
        CAP_C["Kondensator pomiarowy C\n(C0G/NP0 10–22 nF)"]:::circuitStyle
        GND["Masa Układu GND"]:::pwrStyle
    end

    PC["Stanowisko PC (aplikacja pomiarowa)"]:::pcStyle
    PC <-->|"Kabel USB (Dane + 5V)"| InterfaceSection
    LDO -.->|"Szyna zasilania 3.3V"| CP2105
    
    CP2105 -->|"Pin 15 (DTR / GPIO.0)"| RST_STM
    CP2105 -->|"Pin 23 (DTR / GPIO.1)"| RST_AVR
    CP2105 -->|"Pin 11 (RTS) / Pin 14 (GPIO)"| MUX_JUMP

    CP2105 <-->|"UART ECI (TXD/RXD)"| STM32
    CP2105 <-->|"UART SCI (TXD/RXD)"| AVR

    RST_STM -->|"NRST (Wymuszenie High-Z)"| STM32
    RST_AVR -->|"RESET (Wymuszenie High-Z)"| AVR

    MUX_JUMP -->|"Sygnał MUX_SEL"| MUX
    STM32 ==>|"5 linii I/O (TIM_CH1 + 4x GPIO DRV)"| MUX
    AVR ==>|"5 linii I/O (ICP1 + 4x GPIO DRV)"| MUX

    RX --- SUM_NODE
    RC1 --- SUM_NODE
    RC2 --- SUM_NODE
    SHORT --- SUM_NODE

    SUM_NODE -->|"Prąd rozładowania"| R0
    R0 --> VO_NODE
    VO_NODE --- CAP_C
    CAP_C --- GND

    MUX -->|"DRV_RX"| RX
    MUX -->|"DRV_RC1"| RC1
    MUX -->|"DRV_RC2"| RC2
    MUX -->|"DRV_SHORT"| SHORT
    MUX ==>|"CAP (Bezpośredni odczyt Vo)"| VO_NODE

    classDef pcStyle stroke:#64748b,stroke-width:2px;
    classDef pwrStyle stroke:#ef4444,stroke-width:2px;
    classDef commStyle stroke:#3b82f6,stroke-width:2px;
    classDef rstStyle stroke:#d946ef,stroke-width:2px;
    classDef mcuStyle stroke:#22c55e,stroke-width:2px;
    classDef muxStyle stroke:#f59e0b,stroke-width:2px;
    classDef circuitStyle stroke:#a855f7,stroke-width:2px;
    classDef nodeStyle stroke:#94a3b8,stroke-width:2px;
```

### 3.2. Tor Pomiarowy (5 Linii)

![Calibration circuit concept](./calibration_concept.jpg)

Tor pomiarowy składa się z 5 fizycznych ścieżek połączonych w jeden wspólny węzeł napięciowy $V_o$ przy kondensatorze $C$:

1. **Linia `CAP`:** Połączona bezpośrednio z kondensatorem $C$ i wejściem *Input Capture* timera.
2. **Linia `DRV_RX`:** Dołączona do badanego czujnika/rezystora $R_x$.
3. **Linia `DRV_RC1`:** Dołączona do dolnego rezystora wzorcowego $R_{c1}$ (np. $200\ \Omega$, 0.1%).
4. **Linia `DRV_RC2`:** Dołączona do górnego rezystora wzorcowego $R_{c2}$ (np. $10\text{ k}\Omega$, 0.1%).
5. **Linia `DRV_SHORT`:** Dołączona do ścieżki zwarciowej i szeregowego rezystora $R_0$ ograniczającego prąd pinu.

- **Kondensator $C$:** najlepiej typu C0G/NP0 (lub foliowy FKP/MKP) ze względu na niski współczynnik napięciowy i temperaturowy.
- **Gniazda testowe (footprint hybrydowy):** Pola lutownicze SMD połączone z otworami THT, w które wlutowane będą **złocone gniazda kołkowe (Machined Pins)**. Umożliwia to wygodną wymianę elementów $R_x, C, R_{c1}, R_{c2}, R_0$.

### 3.3. Mikrokontrolery Testowe

- **MCU A (8-bit AVR): ATmega328P-AU**
  - Zasilanie: $3.3\text{ V}$, Zegar: $8\text{ MHz}$ kwarcowy ($T_{ref} = 125\text{ ns}$).
  - Interfejs TDC: Sprzętowy moduł *Input Capture* na pinie `ICP1`.
- **MCU B (32-bit ARM): STM32L073RZ (LQFP-64)**
  - Zasilanie: $3.3\text{ V}$, Zegar: $32\text{ MHz}$ ($T_{ref} = 31.25\text{ ns}$).
  - Interfejs TDC: Timery 16-bit z modułem *Input Capture* i optymalizacją pod niski pobór energii (*Stop/Sleep Mode*).

### 3.4. Klucz Analogowy i Izolacja Sprzętowa

- **Scalony Przełącznik:** **TS3A27518E** (6-kanałowy przełącznik 1:2, $r_{on} \approx 4.4\ \Omega$).
  - Łączy 5 linii toru pomiarowego (`CAP`, `DRV_RX`, `DRV_RC1`, `DRV_RC2`, `DRV_SHORT`) albo z STM32, albo z ATmegą.
  - Wejście wyboru `IN1+IN2` posiada rezystor pull-down $10\text{ k}\Omega$ do masy (wymusza domyślny, zdefiniowany stan szyny pomiarowej).
- **Hardware RESET:**
  - Linie RESET obu mikrokontrolerów są niezależnie podciągnięte rezystorami $10\text{ k}\Omega$ do $+3.3\text{ V}$.
  - Połączenie z mostkiem CP2105 przez **diody Schottky'ego (BAT54)** (katody skierowane w stronę CP2105, anody do linii RESET MCU). Zapewnia to bezkonfliktowe ściąganie linii do masy przy jednoczesnym braku kolizji z zewnętrznymi programatorami.
  - Linia `NRST` mikrokontrolera STM32 podłączona jest do **pinu 15 CP2105** (`GPIO.0_ECI` / `DTR_ECI`).
  - Linia `RESET` mikrokontrolera AVR podłączona jest do **pinu 23 CP2105** (`GPIO.1_SCI` / `DTR_SCI`).
  - Nieaktywny mikrokontroler jest trzymany w stanie sprzętowego RESETu, co wymusza stan wysokiej impedancji na wszystkich jego pinach I/O i eliminuje wpływ na wspólny tor pomiarowy.

### 3.5. Zasilanie i Komunikacja USB

- **Zasilanie:** Pobierane z $V_{BUS}$ USB ($5\text{ V}$), stabilizowane przez niskoszumowy regulator LDO na **$3.3\text{ V}$** (np. `MCP1700-3302E` / `AP7361C`) z pojemnościami odsprzęgającymi $100\text{ nF} + 10\ \mu\text{F}$ przy każdym układzie scalonym.
- **Mostek USB:** **CP2105** (Silicon Labs, obudowa QFN-24):
  - **Kanał ECI UART:** Połączony z mikrokontrolerem STM32L073RZ.
  - **Kanał SCI UART:** Połączony z mikrokontrolerem ATmega328P.
- **Dualna architektura sterowania Resetem i Kluczem (Podstawowa / Backup):**
  - Układ PCB można zaprojektować tak, aby umożliwić dwie wymienne metody kontroli bez modyfikacji ścieżek:
    1. **Opcja Podstawowa (linie modemowe UART):**
       - Linie `DTR_ECI` (pin 15) oraz `DTR_SCI` (pin 23) sterują resetami mikrokontrolerów z poziomu standardowej biblioteki portu szeregowego (PySerial: `ser.dtr`).
       - Linia `RTS_ECI` (pin 11) steruje przełączaniem klucza TS3A27518E (`ser.rts`).
    2. **Opcja Rezerwowa (niezależne GPIO):**
       - Te same fizyczne piny 15 (`GPIO.0_ECI`) i 23 (`GPIO.1_SCI`) skonfigurowane w trybie Open-Drain sterują resetami przez API układu CP2105.
       - Linia `GPIO.1_ECI` (pin 14) steruje kluczem analogowym.
  - **Punkt konfiguracyjny sygnału wyboru MUX (`MUX_SEL`):** Na płytce PCB można umieścić zworkę 1×3, pozwalającą połączyć wejście `IN1+IN2` klucza TS3A27518E albo z pinem 11 (`RTS_ECI`), albo z pinem 14 (`GPIO.1_ECI`).

## 5. Software PC

- **Architektura:** Skrypt w Pythonie, który zarządza całą sekwencją pomiarową, zapisuje i przetwarza odebrane dane.
- **Automatyzacja pomiarów i przełączanie MCU:**
  - Skrypt realizuje sekwencję przełączania:
    1. Wymuszenie stanu RESET na obu mikrokontrolerach.
    2. Przełączenie klucza analogowego TS3A27518E na docelowy tor.
    3. Zwolnienie z RESETu wyłącznie mikrokontrolera aktywnego.
  - Wybór mikrokontrolera i sterowanie resetem odbywa się poprzez mostek CP2105 (liniami DTR/RTS lub rejestrami GPIO).
- **Przetwarzanie danych:** Skrypt odbiera zawsze surowe dane i sam dokonuje obliczeń.
