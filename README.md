# esphome-components (wM-Bus / SX1262)

Komponent do bezprzewodowego odczytu liczników wM-Bus (wodomierze, gazomierze, ciepłomierze itp.) w środowisku ESPHome z wykorzystaniem wydajnego frameworka **ESP-IDF** oraz układów radiowych **SX1262**.

*Fork bazuje na kodzie wM-Bus v5 (SzczepanLeon / IoTLabs-pl).*

---

## ⚠️ Wymaganie dla ESPHome 2026.7.0+

Od wersji **ESPHome 2026.7.0** przy korzystaniu z frameworka `esp-idf` należy w sekcji `esp32:` obowiązkowo dodać parametr **`toolchain: platformio`**. 
Jest to wymagane, aby kompilator prawidłowo połączył wszystkie pliki źródłowe biblioteki `wmbusmeters` podczas budowania firmware'u:

```yaml
esp32:
  board: esp32-s3-devkitc-1
  variant: esp32s3
  toolchain: platformio   # <--- Wymagane od ESPHome 2026.7.0 dla esp-idf
  framework:
    type: esp-idf
```

---

## 🚀 Przykładowa konfiguracja (ESP32-S3 + SX1262 + Octal PSRAM)

Kompletna konfiguracja dla płytki **ESP32-S3 (N16R8)** z zewnętrzną anteną na przełączniku **LR30**, obsługującej radio **SX1262** po magistrali SPI oraz odczytującej dane z wodomierzy **Apator 162**:

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
  toolchain: platformio
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
      url: https://github.com/Kuj0n/esphome-components
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

## 📡 Parametry modułu radiowego SX1262

Wszystkie linie magistrali SPI (`clk_pin`, `miso_pin`, `mosi_pin`) konfiguruje się w standardowym bloku `spi:`. W bloku `wmbus_radio:` podaje się piny sterujące samym modułem radiowym:

| Parametr | Wymagany | Opis |
| :--- | :---: | :--- |
| `radio_type` | **tak** | Typ radia – zawsze ustawione na `SX1262`. |
| `cs_pin` | **tak** | Pin SPI Chip Select (CS / NSS). |
| `irq_pin` | **tak** | Pin przerwania (połączony z pinem DIO1 na module). |
| `reset_pin` | **tak** | Pin sprzętowego resetu modułu (RST / NRST). |
| `busy_pin` | nie | Pin sygnału BUSY. Bardzo zalecany dla stabilności komunikacji. |
| `rx_gain` | nie | Tryb czułości odbiornika: `BOOSTED` (domyślny, maksymalny zasięg) lub `POWER_SAVING`. |
| `rf_switch` | nie | Ustaw na `true`, jeśli linia DIO2 bezpośrednio steruje przełącznikiem antenowym na płytce. |
| `has_tcxo` | nie | Ustaw na `true`, jeśli moduł posiada oscylator TCXO zasilany z pinu DIO3. |
