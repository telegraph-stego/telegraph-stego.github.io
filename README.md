# · ТЕЛЕГРАФ · / · TELEGRAPH ·

Стеганографический мессенджер коротких сообщений / Steganographic short message messenger

---

[Русский](#русский) | [English](#english)

---

## Русский

### Что это

Одностраничный мессенджер — **один файл `index.html`**, ноль серверов, ноль аккаунтов, ноль зависимостей. Сообщения передаются через публичные MQTT-брокеры и маскируются под телеметрию IoT-датчиков. Топик меняется каждые 5 минут (FHSS). Лента хранит последние 50 сообщений.

Закрыл вкладку — всё исчезло. Нигде ничего не хранится.

### Зачем

Иногда нужен приватный канал связи без регистрации, без установки приложений и без следов. Телеграф — это цифровой аналог бумажной записки: прочитал, уничтожил.

### Как работает

```
Клиент A                              Клиент B
   │                                      │
   │  seed: "волга-22-закат"              │  seed: "волга-22-закат"
   │         │                            │         │
   │    SHA-256 → профиль + FHSS-топик    │    SHA-256 → профиль + FHSS-топик
   │         │                            │         │
   │    Стеганокодер                      │    Стеганокодер
   │    (текст → IoT JSON)                │    (IoT JSON → текст)
   │         │                            │         ▲
   │         ▼          WSS (TLS)         │         │
   │    MQTT publish ──► broker ─────► MQTT subscribe
   │                                      │
   │    ──── каждые 5 мин: новый топик ────
   │                                      │
   └──────────────────────────────────────┘
```

### Быстрый старт

**Подготовка (один раз, при личной встрече):**

1. Договоритесь с собеседником о кодовой фразе и брокере (например: «волга-22-закат», режим АВТО)
2. Передайте ему файл `index.html` (на флешке, по почте, как угодно)
3. Убедитесь что время на устройствах обоих собеседников точное ([time.is](https://time.is))

**Сеанс связи:**

1. Оба откройте `index.html` в браузере
2. Введите одну и ту же кодовую фразу
3. Придумайте себе позывной (его увидит собеседник)
4. Режим АВТО — нажмите «Подключиться». Режим ВЫБОР — нажмите кнопку согласованного цвета
5. Дождитесь зелёного индикатора и появления позывного собеседника
6. Пишите — до 128 символов на сообщение
7. Закончили — нажмите [✕] или закройте вкладку

**Режим связи:**

Телеграф работает как рация — нет уведомлений и всплывающих сообщений. Чат держится открытым постоянно, пока есть связь. Критичен первый факт подключения — время первого сеанса согласуется вне Телеграфа. При обрыве связи — переподключение в первую минуту каждого часа, ожидание 3 минуты. Если собеседник не появился — повтор в начале следующего часа.

### Важно

- **Оба онлайн одновременно.** Если собеседник не подключен — сообщение теряется безвозвратно. Как рация.
- **Один брокер.** Оба собеседника должны использовать один и тот же брокер. Договоритесь заранее.
- **Кодовую фразу — только лично.** Не передавайте через интернет.
- **Точное время.** Часы на устройствах обоих собеседников должны быть точными (допуск ±1 минута). Проверьте перед сеансом: [time.is](https://time.is)
- **Нет истории.** При нажатии кнопки [✕] чат закрывается, переписка удаляется.
- **Два собеседника.** Один канал = один seed = два человека. Третий участник детектируется автоматически. Для сети — каждый участник открывает несколько вкладок с разными seed (ретрансляция).
- **Свой брокер.** Для максимальной приватности поднимите свой MQTT-брокер:
  ```bash
  docker run -d --name mqtt -p 9001:9001 eclipse-mosquitto
  ```

### Сервисные функции

- `Ctrl+T` — самотестирование (24 теста, проверка всех компонентов)
- `Ctrl+D` — диагностическая панель (топик, FHSS-слот, пакеты, статус)
- Двойной клик / длинный тап — копирование сообщения

### Как это защищает

| Угроза | Защита |
|--------|--------|
| Провайдер читает трафик | TLS (WSS) — содержимое зашифровано на транспортном уровне |
| Наблюдатель видит, что это чат | Стеганография — пакеты выглядят как IoT-телеметрия |
| Подбор топика | SHA-256 от seed — 2²⁵⁶ вариантов, перебор невозможен |
| Сигнатурный анализ формата | Профиль генерируется из seed — каждая пара имеет уникальный формат |
| Наблюдение за топиком | FHSS — топик меняется каждые 5 минут |
| Третий участник | Автоматическая детекция по sid — предупреждение о компрометации |
| Ретроспективный анализ | Нет хранения — ни на клиенте, ни на брокере, ни у провайдера |
| Блокировка сервиса | Нет центрального сервера, любой MQTT-брокер подходит |

### Чего это НЕ делает

- Не защищает от компрометации кодовой фразы
- Не защищает от кейлоггера или камеры за спиной
- Не защищает от целенаправленного анализа конкретного MQTT-брокера
- **Не является заменой Signal, WhatsApp или другим защищённым мессенджерам**

Это инструмент для обычных людей, которым нужен простой приватный канал без регистрации и следов.

### Технические детали

См. [TECHNICAL.md](TECHNICAL.md) — полная архитектура, протокол, модель угроз.

### Правовая информация

Данное ПО носит экспериментальный исследовательский характер, не преследует коммерческих целей и реализует идеи, описанные в открытой научной литературе.

Не является средством шифрования (ПКЗ-2005). Не является организатором распространения информации (ФЗ-149). Не осуществляет хранение данных (ФЗ-374). Использует стандартный протокол MQTT и штатное TLS-шифрование на уровне WebSocket. Ответственность за содержание сообщений несут пользователи.

---

## English

### What is this

A single-page messenger — **one `index.html` file**, zero servers, zero accounts, zero dependencies. Messages are transmitted through public MQTT brokers and disguised as IoT sensor telemetry. Topic changes every 5 minutes (FHSS). Feed keeps last 50 messages.

Close the tab — everything is gone. Nothing is stored anywhere.

### Quick start

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

### Important

- **Both must be online simultaneously.** If your contact is offline, the message is lost forever.
- **Same broker.** Both contacts must use the same broker. Agree in advance.
- **Share the code phrase in person only.** Never over the internet.
- **Accurate clocks.** Both devices must have accurate time (±1 minute). Check: [time.is](https://time.is)
- **No history.** Clicking [✕] closes the chat and deletes all messages.
- **Two people per channel.** Third participant is detected automatically. For networks — relay via multiple tabs.

### How it protects

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

### Legal

This software is experimental and research-oriented, pursuing no commercial goals. It implements ideas described in open scientific literature. It does not implement application-level encryption. Uses standard MQTT protocol with standard TLS at WebSocket transport layer. Users are responsible for the content of their messages.

---

## License

MIT — see [LICENSE](LICENSE)
