# Техническая спецификация / Technical Specification

[Русский](#архитектура) | [English](#architecture)

---

## Архитектура

### Единственный файл

Всё приложение — один файл `index.html` (~300KB с встроенным mqtt.js):

```
index.html
├── <style>                  CSS (терминальная эстетика)
├── <div id="app">           Точка монтирования
├── <script id="mqtt-inline"> mqtt.js v5.10.4 (встроен, без CDN)
└── <script>                 Основной код (IIFE)
    ├── Константы            HEARTBEAT_INTERVAL, FHSS_INTERVAL, ...
    ├── _randomHex()         Общий хелпер
    ├── generateProfile()    Автогенерация профиля из seed
    ├── TopicGenerator       SHA-256 → MQTT-топик (с FHSS)
    ├── StegoCodec           Кодирование/декодирование IoT-пакетов
    ├── Transport            Обёртка над MQTT.js (без fallback)
    ├── Telegraph            Контроллер (FHSS + heartbeat + presence)
    ├── UI                   Два экрана + каты + брокер-селектор
    ├── selfTest()           24 встроенных теста
    └── main()               Точка входа
```

### Зависимости

Ноль внешних зависимостей. mqtt.js v5.10.4 встроен в HTML. Файл полностью автономен.

Используемые API браузера:
- `crypto.subtle.digest('SHA-256')` — генерация топика, профиля и FHSS
- `TextEncoder` / `TextDecoder` — UTF-8
- `AudioContext` / `OscillatorNode` — звуковые уведомления
- `navigator.clipboard.writeText` — копирование сообщений

---

## Протокол

### Автогенерация профиля из seed

Из `SHA-256(seed + "/profile")` детерминированно выбираются все параметры маскировки:

| Параметр | Назначение | Пул значений |
|----------|-----------|-------------|
| Имена сенсорных полей (5) | Ключи в JSON | t/tmp/tc/temp/t_c, h/rh/hm/hum/rh_pct, ... |
| Имя поля payload | Где прячется сообщение | raw/data/buf/dump/diag/payload/bin/dat |
| Префикс clientId | Идентификация MQTT-клиента | iot_/gw_/dev_/node_/edge_/hub_ |
| Префикс устройства | Имя «датчика» | sens_/dev_/nd_/mod_/unit_/probe_ |
| Шаблон топика | Структура MQTT-топика | dt/telemetry/data/msg/iot + /%r/%f/%z/%d |
| Диапазоны значений | Реалистичные границы | Сдвиг ±5 от базовых |

Два клиента с одним seed — идентичный профиль. С разным — разный. Каждая пара создаёт уникальный формат пакетов.

### FHSS — ротация топиков по таймеру

Топик меняется каждые 5 минут, синхронно по системным часам:

```
timeSlot = Math.floor(Date.now() / 300000)
topic = generateTopic(SHA-256(seed + "/" + timeSlot), PROFILE)
```

Подписка на два топика: текущий + предыдущий (покрывает рассинхрон до 5 мин). Публикация — всегда в текущий.

**Требование:** часы точны ±1 минута. Рассинхрон >5 минут = потеря связи.

### Генерация топика

```
seed + "/" + timeSlot → SHA-256 → hex (64 символа)

hex[0:4]   → суффикс устройства
hex[4:6]   → индекс региона (mod длины массива)
hex[6:8]   → номер объекта (mod 99 + 1)
hex[8:10]  → индекс зоны (mod длины массива)

Топик: "{шаблон из PROFILE}" с подстановкой %r, %f, %z, %d
Пример: "telemetry/eu-central/loc31/hvac/nd_e0b7"
```

### Формат пакетов на проводе

Три типа (имена полей зависят от профиля):

**Сообщение** (есть payload):
```json
{"d":"nd_e0b7","t_c":22.53,"hum":60.88,"p":1013.31,"pwr":3.83,"rf":-68,
 "seq":143,"ts":1739620820,"sid":"f3a1b2","payload":"7b2263...7d"}
```

**Heartbeat** (нет payload, есть tag):
```json
{"d":"nd_e0b7","t_c":22.53,"hum":60.88,"p":1013.31,"pwr":3.83,"rf":-68,
 "seq":143,"ts":1739620820,"sid":"f3a1b2","tag":"d09ed180d191d0bb"}
```

**Offline** (st: 0):
```json
{"d":"nd_e0b7","st":0,"sid":"f3a1b2","tag":"d09ed180d191d0bb"}
```

**Декоративные поля** имитируют реальный датчик: значения дрейфуют (±0.05 база, ±0.3 шум), `seq` — автоинкремент, `ts` — Unix timestamp, `sid` — 6 hex символов для фильтрации собственных пакетов.

**Поле payload:** hex-encoded UTF-8 строка JSON: `{"c":"позывной","m":"текст","n":номер}`

**Поле tag** (в heartbeat/offline): hex-encoded позывной.

### Кодирование / декодирование

```
Отправка:  {c, m, n} → JSON → UTF-8 → hex → поле payload
Приём:     поле payload → hex → UTF-8 → JSON → {c, m, n}
Heartbeat: callsign → UTF-8 → hex → поле tag
Тип:       есть payload → message | нет payload, нет st:0 → heartbeat | st:0 → offline
```

---

## Heartbeat и присутствие

- Heartbeat каждые 20 сек (HEARTBEAT_INTERVAL)
- LWT: offline-пакет регистрируется при подключении
- Таймаут: 45 сек (PEER_TIMEOUT)
- Состояния: «ожидание собеседника...» → «{позывной}: ● на связи» → «{позывной}: ○ нет связи»

## Детекция третьего участника

При получении пакета с sid ≠ собственному и ≠ peerSid:
- Красное предупреждение: «⚠ ОБНАРУЖЕН ТРЕТИЙ УЧАСТНИК! КАНАЛ СКОМПРОМЕТИРОВАН.»
- Показывается один раз. Автоматическое отключение не производится.

---

## Транспорт

**Протокол:** MQTT v3.1.1 over WSS
**Библиотека:** mqtt.js v5.10.4, встроена в HTML

**Два режима подключения:**

| Режим | Поведение |
|-------|-----------|
| АВТО | HiveMQ (wss://broker.hivemq.com:8884/mqtt). Всегда. Без fallback. |
| ВЫБОР | Три цветные кнопки + свой URL. Без fallback. |

Цвета: синий (#4488ff) — HiveMQ, зелёный (#44bb44) — EMQX, оранжевый (#ff8844) — Mosquitto.

**Параметры:** QoS 0, retain false, clean session true, keepalive 60с, таймаут 10с.
**Fallback удалён.** Автопереподключение только к тому же брокеру.

---

## Модель угроз

### Защита

| Механизм | От чего защищает |
|----------|-----------------|
| TLS (WSS) | Пассивный перехват провайдером |
| Стеганография | Идентификация трафика как мессенджера |
| SHA-256 от seed | Перебор топика (2²⁵⁶ вариантов) |
| Автогенерация профиля | Сигнатурный анализ формата |
| FHSS | Наблюдение за одним топиком |
| Heartbeat | Маскировка паттерна трафика |
| Детекция третьего | Компрометация канала |
| Нулевое хранение | Ретроспективный анализ |
| Лимит 50 сообщений | Накопление данных в памяти |
| Некликабельные ссылки | Утечки через DNS/HTTP |
| Децентрализация | Блокировка сервиса |

### Ограничения

| Угроза | Почему не защищает |
|--------|-------------------|
| Компрометация seed | Вся коммуникация предсказуема |
| Endpoint compromise | Кейлоггер, скриншот, камера |
| Корреляция по времени | Два «устройства» синхронно появляются/исчезают |
| Wildcard + парсер | Подписка на `#` с анализом payload |
| Брокер ведёт логи | Маловероятно, но возможно |

---

## Юридический анализ (РФ)

**ПКЗ-2005:** Не реализует шифрование на прикладном уровне. TLS — штатное средство браузера.

**ФЗ-149:** Не является ОРИ — нет серверной части.

**ФЗ-374:** Не хранит данные. Обязательства по хранению — на провайдере.

**Стеганография:** Не является предметом правового регулирования в РФ.

ПО носит экспериментальный исследовательский характер, не преследует коммерческих целей, реализует идеи из открытой научной литературы.

---

## Свой брокер (Docker)

```bash
mkdir -p mosquitto/config
cat > mosquitto/config/mosquitto.conf << 'EOF'
listener 9001
protocol websockets
allow_anonymous true
EOF

docker run -d --name telegraph-broker -p 9001:9001 \
  -v $(pwd)/mosquitto/config:/mosquitto/config eclipse-mosquitto
```

В приложении: ВЫБОР → «свой...» → `ws://IP:9001/mqtt`

---

## Сетевая архитектура

Один канал = один seed = два собеседника. Для сети — ретрансляция:

```
A ←seed1→ B ←seed2→ C ←seed3→ D
```

Каждый сегмент независим. Компрометация одного не раскрывает остальные. Принцип минимального знания.

---

## Self-test (24 теста)

PF (2), TG (4), SC (6), ENC (3), SC cross (1), FHSS (5), HB (3)

---
---

# Technical Specification (English)

## Architecture

### Single file

The entire application is one `index.html` file (~300KB with embedded mqtt.js):

```
index.html
├── <style>                  CSS (terminal aesthetic)
├── <div id="app">           Mount point
├── <script id="mqtt-inline"> mqtt.js v5.10.4 (embedded, no CDN)
└── <script>                 Main code (IIFE)
    ├── Constants            HEARTBEAT_INTERVAL, FHSS_INTERVAL, ...
    ├── _randomHex()         Shared helper
    ├── generateProfile()    Auto-generate masking profile from seed
    ├── TopicGenerator       SHA-256 → MQTT topic (with FHSS)
    ├── StegoCodec           Encode/decode IoT packets
    ├── Transport            MQTT.js wrapper (no fallback)
    ├── Telegraph            Controller (FHSS + heartbeat + presence)
    ├── UI                   Two screens + collapsibles + broker selector
    ├── selfTest()           24 built-in tests
    └── main()               Entry point
```

### Dependencies

Zero external dependencies. mqtt.js v5.10.4 is embedded directly in HTML. The file is fully autonomous.

Browser APIs used:
- `crypto.subtle.digest('SHA-256')` — topic, profile, and FHSS generation
- `TextEncoder` / `TextDecoder` — UTF-8
- `AudioContext` / `OscillatorNode` — audio alerts
- `navigator.clipboard.writeText` — message copying

---

## Protocol

### Auto-generated profile from seed

All masking parameters are deterministically derived from `SHA-256(seed + "/profile")`:

| Parameter | Purpose | Pool |
|-----------|---------|------|
| Sensor field names (5) | JSON keys | t/tmp/tc/temp/t_c, h/rh/hm/hum/rh_pct, ... |
| Payload field name | Where the message hides | raw/data/buf/dump/diag/payload/bin/dat |
| Client ID prefix | MQTT client identification | iot_/gw_/dev_/node_/edge_/hub_ |
| Device prefix | "Sensor" name | sens_/dev_/nd_/mod_/unit_/probe_ |
| Topic template | MQTT topic structure | dt/telemetry/data/msg/iot + /%r/%f/%z/%d |
| Value ranges | Realistic sensor boundaries | ±5 shift from base |

Two clients with the same seed produce identical profiles. Different seeds produce different profiles. Each pair creates a unique packet format undetectable by another pair's signature.

### FHSS — timer-based topic rotation

The topic changes every 5 minutes, synchronized by system clocks:

```
timeSlot = Math.floor(Date.now() / 300000)
topic = generateTopic(SHA-256(seed + "/" + timeSlot), PROFILE)
```

Both clients subscribe to current + previous topics (covers clock skew up to 5 min). Publishing is always to current.

**Requirement:** clocks accurate ±1 minute. Skew >5 minutes = loss of communication.

### Topic generation

```
seed + "/" + timeSlot → SHA-256 → hex (64 chars)

hex[0:4]   → device suffix
hex[4:6]   → region index (mod array length)
hex[6:8]   → facility number (mod 99 + 1)
hex[8:10]  → zone index (mod array length)

Topic: "{template from PROFILE}" with %r, %f, %z, %d substitution
Example: "telemetry/eu-central/loc31/hvac/nd_e0b7"
```

### Wire format

Three packet types (field names depend on profile):

**Message** (has payload):
```json
{"d":"nd_e0b7","t_c":22.53,"hum":60.88,"p":1013.31,"pwr":3.83,"rf":-68,
 "seq":143,"ts":1739620820,"sid":"f3a1b2","payload":"7b2263...7d"}
```

**Heartbeat** (no payload, has tag):
```json
{"d":"nd_e0b7","t_c":22.53,"hum":60.88,"p":1013.31,"pwr":3.83,"rf":-68,
 "seq":143,"ts":1739620820,"sid":"f3a1b2","tag":"d09ed180d191d0bb"}
```

**Offline** (st: 0):
```json
{"d":"nd_e0b7","st":0,"sid":"f3a1b2","tag":"d09ed180d191d0bb"}
```

**Decorative fields** simulate a real sensor: values drift (±0.05 base, ±0.3 noise), `seq` auto-increments, `ts` is Unix timestamp, `sid` is 6-hex session ID for self-filtering.

**Payload field:** hex-encoded UTF-8 JSON: `{"c":"callsign","m":"text","n":number}`

**Tag field** (heartbeat/offline): hex-encoded callsign.

### Encoding / decoding

```
Send:      {c, m, n} → JSON → UTF-8 → hex → payload field
Receive:   payload → hex → UTF-8 → JSON → {c, m, n}
Heartbeat: callsign → UTF-8 → hex → tag field
Type:      has payload → message | no payload, no st:0 → heartbeat | st:0 → offline
```

---

## Heartbeat and presence

- Heartbeat every 20s (HEARTBEAT_INTERVAL)
- LWT: offline packet registered on connect
- Timeout: 45s (PEER_TIMEOUT)
- States: "waiting for peer..." → "{callsign}: ● online" → "{callsign}: ○ offline"

## Third participant detection

On receiving packet with sid ≠ own and ≠ peerSid:
- Red warning: "⚠ THIRD PARTICIPANT DETECTED! CHANNEL COMPROMISED."
- Shown once. No automatic disconnect.

---

## Transport

**Protocol:** MQTT v3.1.1 over WSS
**Library:** mqtt.js v5.10.4, embedded in HTML

**Two connection modes:**

| Mode | Behavior |
|------|----------|
| AUTO | HiveMQ (wss://broker.hivemq.com:8884/mqtt). Always. No fallback. |
| SELECT | Three colored buttons + custom URL. No fallback. |

Colors: blue (#4488ff) — HiveMQ, green (#44bb44) — EMQX, orange (#ff8844) — Mosquitto.

**Parameters:** QoS 0, retain false, clean session true, keepalive 60s, timeout 10s.
**Fallback removed.** Auto-reconnect only to same broker.

---

## Threat model

### Protection

| Mechanism | Protects against |
|-----------|-----------------|
| TLS (WSS) | Passive traffic interception by ISP |
| Steganography | Identifying traffic as a messenger |
| SHA-256 of seed | Topic brute-force (2²⁵⁶ possibilities) |
| Auto-generated profile | Signature-based format analysis |
| FHSS timer | Prolonged surveillance of a single topic |
| Heartbeat | Traffic pattern masking |
| Third-party detection | Channel compromise |
| Zero storage | Retrospective analysis |
| 50-message limit | Memory accumulation |
| Non-clickable links | DNS/HTTP leaks |
| Decentralization | Service blocking |

### Limitations

| Threat | Why unprotected |
|--------|----------------|
| Seed compromise | All communication becomes predictable |
| Endpoint compromise | Keylogger, screenshot, shoulder surfing |
| Timing correlation | Two "devices" appear/disappear synchronously |
| Wildcard + parser | Subscribe `#` on broker with payload analysis |
| Broker logging | Unlikely for public brokers, but possible |

---

## Legal analysis (Russian Federation)

**PKZ-2005:** No application-level encryption. TLS is a standard browser facility.

**FZ-149:** Not an information distribution organizer — no server component.

**FZ-374:** Does not store data. Storage obligations fall on the ISP.

**Steganography:** Not a subject of legal regulation in the Russian Federation.

The software is experimental and research-oriented, pursuing no commercial goals. It implements ideas described in open scientific literature.

---

## Self-hosted broker (Docker)

```bash
mkdir -p mosquitto/config
cat > mosquitto/config/mosquitto.conf << 'EOF'
listener 9001
protocol websockets
allow_anonymous true
EOF

docker run -d --name telegraph-broker -p 9001:9001 \
  -v $(pwd)/mosquitto/config:/mosquitto/config eclipse-mosquitto
```

In the app: SELECT → "custom..." → `ws://YOUR-IP:9001/mqtt`

---

## Network architecture

One channel = one seed = two users. For networks — relay:

```
A ←seed1→ B ←seed2→ C ←seed3→ D
```

Each segment is independent. Compromising one does not reveal others. Principle of least knowledge.

---

## Self-test (24 tests)

PF (2), TG (4), SC (6), ENC (3), SC cross (1), FHSS (5), HB (3)
