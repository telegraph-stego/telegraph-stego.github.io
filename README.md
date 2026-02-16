# · TELEGRAPH ·

Steganographic short message messenger

---

## What is this

A single-page messenger — **one `index.html` file**, zero servers, zero accounts, zero dependencies. Messages are transmitted through public MQTT brokers and disguised as IoT sensor telemetry. Topic changes every 5 minutes (FHSS). Feed keeps last 50 messages.

Close the tab — everything is gone. Nothing is stored anywhere.

## Why

Sometimes you need a private communication channel without registration, without installing apps, and without traces. Telegraph is a digital equivalent of a paper note: read it, destroy it.

## How it works

```
Client A                              Client B
   │                                      │
   │  seed: "volga-22-sunset"             │  seed: "volga-22-sunset"
   │         │                            │         │
   │    SHA-256 → profile + FHSS topic    │    SHA-256 → profile + FHSS topic
   │         │                            │         │
   │    StegoCodec                        │    StegoCodec
   │    (text → IoT JSON)                 │    (IoT JSON → text)
   │         │                            │         ▲
   │         ▼          WSS (TLS)         │         │
   │    MQTT publish ──► broker ─────► MQTT subscribe
   │                                      │
   │    ──── every 5 min: new topic ──────
   │                                      │
   └──────────────────────────────────────┘
```

## Quick start

**Preparation (once, in person):**

1. Agree on a code phrase and broker with your contact (e.g.: "volga-22-sunset", AUTO mode)
2. Give them the `index.html` file (USB drive, email, any way you like)
3. Make sure clocks on both devices are accurate ([time.is](https://time.is))

**Session:**

1. Both open `index.html` in a browser
2. Enter the same code phrase
3. Choose a callsign (your contact will see it)
4. AUTO mode — click "Connect". SELECT mode — click the agreed color button
5. Wait for the green indicator and your contact's callsign
6. Type — up to 128 characters per message
7. Done — click [✕] or close the tab

**Radio discipline:**

Telegraph works like a radio — no notifications, no popups. Keep the chat open as long as you need it. The first connection moment is critical — agree on session time outside Telegraph. On disconnect — reconnect in the first minute of each hour, wait 3 minutes.

## Important

- **Both must be online simultaneously.** If your contact is offline, the message is lost forever.
- **Same broker.** Both contacts must use the same broker. Agree in advance.
- **Share the code phrase in person only.** Never over the internet.
- **Accurate clocks.** Both devices must have accurate time (±1 minute). Check: [time.is](https://time.is)
- **No history.** Clicking [✕] closes the chat and deletes all messages.
- **Two people per channel.** Third participant is detected automatically. For networks — relay via multiple tabs.
- **Self-hosted broker.** For maximum privacy, run your own MQTT broker:
  ```bash
  docker run -d --name mqtt -p 9001:9001 eclipse-mosquitto
  ```

## Service functions

- `Ctrl+T` — self-test (24 tests, all components verified)
- `Ctrl+D` — diagnostic panel (topic, FHSS slot, packets, status)
- Double-click / long-tap — copy message

## How it protects

| Threat | Protection |
|--------|-----------|
| ISP reads traffic | TLS (WSS) — content encrypted at transport level |
| Observer sees it's a chat | Steganography — packets look like IoT telemetry |
| Topic guessing | SHA-256 of seed — 2²⁵⁶ possibilities |
| Signature-based detection | Profile generated from seed — each pair has unique format |
| Topic surveillance | FHSS — topic changes every 5 minutes |
| Third participant | Automatic detection by sid — compromise warning |
| Retrospective analysis | No storage — not on client, not on broker, not at ISP |
| Service blocking | No central server, any MQTT broker works |

## What it does NOT do

- Does not protect against seed compromise
- Does not protect against keyloggers or cameras
- Does not protect against targeted analysis of a specific MQTT broker
- **Is not a replacement for Signal, WhatsApp, or other secure messengers**

This is a tool for ordinary people who need a simple private channel without registration and without traces.

## Localization

Interface supports multiple languages (EN, RU). Language is auto-detected from browser settings. To add a new language: copy the `en` block in the `STRINGS` object, translate, add with a new key (e.g. `de`, `zh`, `es`). The language switcher updates automatically.

## Technical details

See [TECHNICAL.md](TECHNICAL.md) — full architecture, protocol, threat model.

## Legal

This software is experimental and research-oriented, pursuing no commercial goals. It implements ideas described in open scientific literature. It does not implement application-level encryption. Uses standard MQTT protocol with standard TLS at WebSocket transport layer. Users are responsible for the content of their messages.

## License

MIT — see [LICENSE](LICENSE)
