# Technical Specification

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
    ├── STRINGS              Localization (en, ru)
    ├── _randomHex()         Shared helper
    ├── generateProfile()    Auto-generate masking profile from seed
    ├── TopicGenerator       SHA-256 → MQTT topic (with FHSS)
    ├── StegoCodec           Encode/decode IoT packets
    ├── Transport            MQTT.js wrapper (no fallback)
    ├── Telegraph            Controller (FHSS + heartbeat + presence)
    ├── UI                   Two screens + collapsibles + broker selector + i18n
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

## Localization

All UI strings are stored in the `STRINGS` object at the top of the IIFE. Each language is a key (`en`, `ru`) containing all translatable strings.

```javascript
var STRINGS = {
  en: { _name: 'EN', codephrase: 'CODE PHRASE', ... },
  ru: { _name: 'RU', codephrase: 'КОДОВАЯ ФРАЗА', ... }
};
```

**Auto-detection:** `navigator.language` → first two chars → lookup in STRINGS → fallback to `en`.

**Function `t(key)`:** returns `STRINGS[currentLang][key]` → fallback `STRINGS['en'][key]` → fallback `key`.

**DOM binding:** elements with `data-i18n="key"` are updated by `updateUI()`. Dynamic strings (status, errors, warnings) use `t()` at generation time.

**Adding a language:** copy the `en` block, translate values, add with a new key. The switcher generates automatically from `Object.keys(STRINGS)`.

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

## Legal analysis

**Encryption:** No application-level encryption. TLS is a standard browser facility.

**Data distribution:** Not an information distribution service — no server component.

**Data retention:** Does not store data. Storage obligations fall on the ISP.

**Steganography:** Not a subject of legal regulation in most jurisdictions. Masking data format is not prohibited.

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
