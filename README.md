# esphome-components (wM-Bus v5)

> **Informacja o projekcie:** Niniejszy fork bazuje bezpośrednio na **wersji v5 od SzczepanLeon** (opartej na pierwotnym kodzie od IoTLabs-pl / Kuby).  
> Komponent zapewnia obsługę odbioru telegramów wM-Bus (m.in. z liczników wody, gazu, prądu czy ciepłomierzy) w środowisku ESPHome z wykorzystaniem wydajnego frameworka **ESP-IDF**.

> **_NOTE:_** Starsze wersje komponentu (z obsługą wyłącznie CC1101 w środowisku Arduino) znajdziesz w oryginalnym repozytorium:  
> [version 4](https://github.com/SzczepanLeon/esphome-components/tree/version_4) | [version 3](https://github.com/SzczepanLeon/esphome-components/tree/version_3) | [version 2](https://github.com/SzczepanLeon/esphome-components/tree/version_2)

---

## ⚠️ Kluczowa uwaga dla ESPHome 2026.7.0+ i ESP-IDF

Od wersji **ESPHome 2026.7.0** domyślnym narzędziem budującym (toolchainem) dla frameworka `esp-idf` stał się natywny CMake (`toolchain: esp-idf`). Wywołuje to dwa krytyczne problemy przy kompilacji tego komponentu na nowym silniku:
1. **Błąd linkera (`undefined reference`):** Natywny skrypt CMake ignoruje pliki źródłowe z rozszerzeniem `.cc` (których używa silnik `wmbusmeters`), przez co funkcje dekodujące nie są w ogóle kompilowane.
2. **Konflikt nazw (`STATUS`):** Nowe nagłówki systemowe ESP-IDF 5.5.4 dla pamięci ROM (zwłaszcza na układach ESP32-S3) rezerwują globalne słowo `STATUS`, co koliduje z wyliczeniem w pliku `meters.h` (`error: 'STATUS' redeclared as different kind of entity`).

**Rozwiązanie:** Aby komponent kompilował się bezproblemowo na najnowszych wersjach ESPHome z zachowaniem pełnej wydajności ESP-IDF, należy w sekcji `esp32:` **obowiązkowo dopisać parametr `toolchain: platformio`**, który przywraca poprawną obsługę plików `.cc` i eliminuje kolizję nagłówków:

```yaml
esp32:
  board: esp32-s3-devkitc-1
  variant: esp32s3
  toolchain: platformio   # <--- KLUCZOWY PARAMETR DLA ESPHOME 2026.7.0+
  framework:
    type: esp-idf
```

---

## 🚀 Przykładowa konfiguracja (ESP32-S3 + SX1262 + Octal PSRAM)

Poniżej znajduje się kompletny, w pełni przetestowany przykład konfiguracji dla płytki **ESP32-S3 (N16R8)** z zewnętrzną anteną na przełączniku **LR30**, obsługującej radio **SX1262** po magistrali SPI oraz odczytującej dane z wodomierzy **Apator 162**:

```yaml
substitutions:
  name: "wmbus-skaner"

esphome:
  name: ${name}
  friendly_name: "wmBus"
  platformio_options:
    upload_speed: 921600

esp32:
  board: esp32-s3-devkitc-1
  variant: esp32s3
  flash_size: 16MB
  toolchain: platformio   # Wymagane od ESPHome 2026.7.0 dla frameworka esp-idf!
  framework:
    type: esp-idf
    sdkconfig_options:
      CONFIG_ESP_MAIN_TASK_STACK_SIZE: "16384"
      CONFIG_ESP_TASK_WDT_TIMEOUT_S: "60"
      CONFIG_SPIRAM: "y"
      CONFIG_SPIRAM_MODE_OCT: "y"

external_components:
  - source:
      type: git
      url: [https://github.com/Kuj0n/esphome-components](https://github.com/Kuj0n/esphome-components)
      ref: main
    components: [ wmbus_common, wmbus_radio, wmbus_meter ]
    refresh: 1d

switch:
  # Programowe sterowanie przełącznikiem antenowym LR30
  - platform: gpio
    pin: GPIO16
    id: lr30_rxen
    name: "LR30 RX Enable"
    restore_mode: ALWAYS_ON
    internal: true

  - platform: gpio
    pin: GPIO17
    id: lr30_txen
    name: "LR30 TX Enable"
    restore_mode: ALWAYS_OFF
    internal: true

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  power_save_mode: LIGHT
  manual_ip:
    static_ip: !secret static_ip_3
    gateway: !secret gateway
    subnet: 255.255.255.0
    dns1: 8.8.8.8
    dns2: !secret dns2

time:
  - platform: homeassistant
    id: sntp_time

logger:
  level: WARN

api:
  encryption:
    key: !secret api_encryption_key_3

ota:
  - platform: esphome
    password: !secret ota_password

# --- Konfiguracja sprzętowego SPI ---
spi:
  clk_pin: GPIO4
  miso_pin: GPIO5
  mosi_pin: GPIO6

# --- Konfiguracja Radia SX1262 ---
wmbus_radio:
  radio_type: SX1262
  cs_pin: GPIO7
  reset_pin: GPIO9
  irq_pin: GPIO8
  busy_pin: GPIO15
  rx_gain: BOOSTED
  rf_switch: false     # Sterowanie anteną odbywa się przez sekcję 'switch' wyżej
  has_tcxo: false      # Zmień na true w przypadku korzystania z zewnętrznego oscylatora TCXO

# --- Konfiguracja Liczników ---
wmbus_meter:
  - id: licznik_glowny
    meter_id: !secret id_licznik_glowny
    type: apator162
    key: !secret klucz_woda
    on_telegram:
      - component.update: stan_domowy
      - text_sensor.template.publish:
          id: ostatni_odczyt_glowny
          state: !lambda 'return id(sntp_time).now().strftime("%H:%M:%S (%d.%m)");'

  - id: licznik_ogrod
    meter_id: !secret id_licznik_ogrod
    type: apator162
    key: !secret klucz_woda
    on_telegram:
      - component.update: stan_domowy
      - text_sensor.template.publish:
          id: ostatni_odczyt_ogrod
          state: !lambda 'return id(sntp_time).now().strftime("%H:%M:%S (%d.%m)");'

text_sensor:
  - platform: template
    id: ostatni_odczyt_glowny
    name: "Licznik Główny - Ostatni odczyt"
    icon: "mdi:clock-outline"

  - platform: template
    id: ostatni_odczyt_ogrod
    name: "Licznik Ogród - Ostatni odczyt"
    icon: "mdi:clock-outline"

debug:

sensor:
  - platform: wmbus_meter
    parent_id: licznik_glowny
    field: total_m3
    id: stan_glowny
    name: "Licznik Główny - Stan"
    accuracy_decimals: 3
    unit_of_measurement: "m³"
    device_class: water
    state_class: total_increasing

  - platform: wmbus_meter
    parent_id: licznik_ogrod
    field: total_m3
    id: stan_ogrod
    name: "Licznik Ogród - Stan"
    accuracy_decimals: 3
    unit_of_measurement: "m³"
    device_class: water
    state_class: total_increasing

  - platform: template
    name: "Licznik Domowy - Stan"
    id: stan_domowy
    accuracy_decimals: 3
    unit_of_measurement: "m³"
    device_class: water
    state_class: total_increasing
    lambda: |-
      if (std::isnan(id(stan_glowny).state) || std::isnan(id(stan_ogrod).state)) {
        return {};
      }
      return id(stan_glowny).state - id(stan_ogrod).state;

  - platform: wmbus_meter
    parent_id: licznik_glowny
    field: rssi_dbm
    name: "Licznik Główny - Sygnał RSSI"
    entity_category: diagnostic

  - platform: wmbus_meter
    parent_id: licznik_ogrod
    field: rssi_dbm
    name: "Licznik Ogród - Sygnał RSSI"
    entity_category: diagnostic

  - platform: debug
    free:
      name: "ESP32 Free RAM"
    min_free:
      name: "ESP32 Min Free RAM"
    loop_time:
      name: "ESP32 Czas Pętli"
    cpu_frequency:
      name: "ESP32 Taktowanie CPU"

  - platform: uptime
    name: "ESP32 Uptime"

  - platform: wifi_signal
    name: "ESP32 WiFi Signal"

  - platform: internal_temperature
    name: "ESP32 CPU Temp"

button:
  - platform: restart
    name: "ESP32 Restart"
    id: restart_button

  - platform: safe_mode
    name: "ESP32 Tryb Awaryjny"

binary_sensor:
  - platform: template
    name: "WMBus Skaner - Czas Zsynchronizowany"
    lambda: 'return id(sntp_time).now().is_valid();'
    device_class: connectivity
    entity_category: diagnostic
```

---

## 📡 Obsługiwane moduły radiowe i ich parametry

### CC1101
Dla klasycznego modułu CC1101 możesz skonfigurować częstotliwość pracy (zazwyczaj 868.95 MHz dla wM-Bus w Europie):

```yaml
wmbus_radio:
  radio_type: CC1101
  frequency: 868.95MHz       # Opcjonalne: zakres 300–928 MHz (domyślnie 868.95 MHz)
  mosi_pin: GPIO11           # Linia SPI
  miso_pin: GPIO13           # Linia SPI
  clk_pin: GPIO12            # Linia SPI
  cs_pin: GPIO10             # Chip Select (CS / SS)
  irq_pin: GPIO5             # Linia przerwania (GDO0 / GD0)
```

| Parametr | Wymagany | Domyślnie | Opis |
| :--- | :---: | :---: | :--- |
| `radio_type` | tak | — | Typ radia: `CC1101` |
| `cs_pin` | tak | — | Pin SPI Chip Select |
| `irq_pin` | tak | — | Pin przerwania (połączony z pinem GDO0 na module) |
| `frequency` | nie | `868.95MHz` | Częstotliwość pracy z zakresu 300–928 MHz |

*Przetestowano m.in. na płytkach ESP32-C3 Super Mini + CC1101 v2.0 (E07-M1101D-SMA - niebieska płytka).*

---

### SX1276
Dla układów SX1276 (np. moduły LoRa/RFM95) konfigurujesz magistralę SPI standardowo w bloku `spi:`, a w bloku radia wskazujesz dodatkowo pin resetu oraz przerwania (jako DIO1). Przerwania wyzwalane są przy niepustym buforze FIFO:

```yaml
wmbus_radio:
  radio_type: SX1276
  cs_pin: GPIO18
  reset_pin: GPIO14
  irq_pin: GPIO26            # Podłączone do DIO1 na module
```

---

### SX1262
Dla nowoczesnych układów SX1262 konfiguracja jest zbliżona do SX1276, ale udostępnia dodatkowe zaawansowane parametry kontrolne:

```yaml
wmbus_radio:
  radio_type: SX1262
  cs_pin: GPIO23
  reset_pin: GPIO4
  irq_pin: GPIO7             # Podłączone do DIO1
  busy_pin: GPIO19           # Opcjonalne, ale bardzo zalecane do prawidłowego timingu
  rx_gain: BOOSTED           # BOOSTED (domyślnie, wyższa czułość) lub POWER_SAVING
  rf_switch: false           # Ustaw na true, jeśli DIO2 bezpośrednio steruje przełącznikiem RF
  has_tcxo: true             # Ustaw na true, jeśli DIO3 zasila zewnętrzny oscylator TCXO
```

**Specyficzne parametry dla SX1262:**
- `busy_pin`: Opcjonalny pin GPIO dla sygnału BUSY z układu SX1262. Wysoce zalecany dla zachowania stabilności transmisji.
- `rx_gain`: Tryb wzmocnienia odbiornika: `BOOSTED` (lepsza czułość, domyślny) lub `POWER_SAVING` (mniejszy pobór prądu).
- `rf_switch`: Ustaw na `true`, jeśli linia DIO2 układu SX1262 jest fizycznie połączona ze sterowaniem przełącznika antenowego TX/RX na płytce modułu.
- `has_tcxo`: Ustaw na `true`, jeśli moduł posiada oscylator TCXO zasilany z pinu DIO3 (np. większość modułów Waveshare/Ebyte).

---

## 📋 Lista zadań (TODO / DONE)

### TODO:
- [ ] Przygotowanie gotowych pakietów (packages) dla popularnych płytek (np. UltimateReader) z wyświetlaczami, diodami LED itp.
- [ ] Agresywne czyszczenie klas i struktur biblioteki wmbusmeters w celu oszczędności pamięci.
- [ ] Refaktoryzacja logów i śledzenia transmisji (traces).

### DONE:
- [x] Dodanie konfiguracji częstotliwości dla CC1101 (300–928 MHz, domyślnie 868.95 MHz).
- [x] Pełna obsługa CC1101 wraz z obsługą przepełnienia bufora FIFO i obejściem błędów sprzętowych (errata workaround).
- [x] Dodanie obsługi radia SX1262 (z ograniczoną długością ramki) oraz SX1276.
- [x] Reużycie mechanizmów CRC i parserów ramek bezpośrednio z wmbusmeters.
- [x] Refaktoryzacja dekodera 3out6.
- [x] Migracja do środowiska `esp-idf` i całkowite porzucenie frameworka Arduino!
- [x] Uruchomienie odbiornika radiowego w dedykowanym, osobnym tasku FreeRTOS.
- [x] Usunięcie zbędnych komponentów niepowiązanych z wM-Bus z części radiowej.
- [x] Możliwość podawania klucza szyfrowania w formacie ASCII.
- [x] Podział architektoniczny kodu na osobne komponenty (`wmbus_radio` dla komunikacji radiowej, `wmbus_meter` dla liczników oraz `wmbus_common` dla rdzenia wmbusmeters).
- [x] Dodanie wyzwalaczy (triggers):
  - Radio -> `on_packet` (pozwala np. mignąć diodą przy odbiorze dowolnej ramki lub telegramu).
  - Meter -> `on_telegram` (pozwala wywołać akcję po odbiorze prawidłowego odczytu ze zdefiniowanego licznika).
