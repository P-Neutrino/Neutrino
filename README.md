# ⚛️ NEUTRINO Protocol

> *"Passes through everything. Undetectable. Unstoppable."*

**Version:** 0.1 (Concept / RFC)
**Status:** Open for implementations
**Language:** Any. Seriously, any.

---

## ⚡ TL;DR для тех, кто понимает

NEUTRINO — это не VPN и не прокси.
Это протокол передачи данных, который **физически невозможно заблокировать**,
потому что он живёт внутри инфраструктуры, без которой интернет перестаёт работать.

Три несущих канала:
- **DNS** — без него не открывается ни один сайт в мире
- **Cloudflare CDN** — ~20% всего интернета, блокировка = смерть банков
- **NTP** — без него недействительны все TLS-сертификаты планеты

Заблокировать NEUTRINO = заблокировать сам интернет.

---

## 📖 Содержание

1. [Почему это нужно](#1-почему-это-нужно)
2. [Почему существующие решения падают](#2-почему-существующие-решения-падают)
3. [Ключевая идея](#3-ключевая-идея)
4. [Архитектура](#4-архитектура)
5. [Канал 1 — DNS Covert Channel](#5-канал-1--dns-covert-channel)
6. [Канал 2 — CDN WebSocket Fronting](#6-канал-2--cdn-websocket-fronting)
7. [Канал 3 — NTP Timing Steganography](#7-канал-3--ntp-timing-steganography)
8. [Multipath Scheduler](#8-multipath-scheduler)
9. [Шифрование](#9-шифрование)
10. [Как выглядит трафик для DPI](#10-как-выглядит-трафик-для-dpi)
11. [Таблица атак и защит](#11-таблица-атак-и-защит)
12. [Пропускная способность](#12-пропускная-способность)
13. [Требования к серверной части](#13-требования-к-серверной-части)
14. [Как сделать свою реализацию](#14-как-сделать-свою-реализацию)
15. [Как контрибьютить](#15-как-контрибьютить)
16. [FAQ](#16-faq)
17. [Лицензия](#17-лицензия)

---

## 1. Почему это нужно

В России заблокированы (список пополняется каждый месяц):

| Сервис | Год | Кто пострадал |
|---|---|---|
| LinkedIn | 2016 | Профессионалы, HR |
| Telegram (1-я блокировка) | 2018–2020 | Все |
| Twitter/X | 2022 | Журналисты, активисты |
| Facebook | 2022 | Малый бизнес, семьи |
| Instagram | 2022 | Блогеры, магазины |
| YouTube | 2024–2026 | 95 млн пользователей |
| Discord | 2024 | Разработчики, геймеры |
| Signal | 2024 | Журналисты, медики, юристы |
| Viber | 2024 | Пенсионеры, семьи |
| Roblox | 2025 | Дети |
| WhatsApp (звонки) | 2025 | Все |
| kernel.org (Linux) | 2026 | Разработчики ОС |
| 469+ VPN-сервисов | 2026 | Все |
| Telegram (2-я попытка) | 2026 | Все |

Аналогичная ситуация в Иране, Китае, Беларуси, ОАЭ, Пакистане.
**4 миллиарда человек** живут под интернет-цензурой.

Каждый раз, когда появлялся новый «незаблокируемый» инструмент — его блокировали.
NEUTRINO создан чтобы это прекратить.

---

## 2. Почему существующие решения падают

### Как работает блокировка (коротко)

РКН использует **ТСПУ** — железо с DPI (Deep Packet Inspection), стоящее у каждого оператора.

Современный DPI умеет:
- Анализировать TLS fingerprint (ClientHello, порядок extensions)
- Детектировать статистические паттерны потока (размеры пакетов, тайминги)
- Применять ML-классификаторы на трафике
- Работать с SNI в TLS 1.3
- Блокировать по поведению соединения, а не по содержимому

### Почему VLESS/XTLS упали

VLESS выглядел как обычный HTTPS — это была его сила.
Но у него есть **статистические паттерны**:

```
Обычный HTTPS браузера:
  → burst запросов → тишина → burst → тишина
  → packet sizes: 100-1500 байт, нерегулярно
  → соотношение up/down: ~1:10 (загружаем больше, чем отправляем)

VLESS туннель:
  → постоянный поток
  → нетипичный размер пакетов (MTU-aligned)
  → соотношение up/down близко к 1:1 при туннелировании
  → нет характерных паттернов браузерных fingerprint'ов
```

DPI не читает содержимое — он смотрит на **поведение**.
Этого достаточно для классификатора.

### Почему Tor медленный и тоже блокируется

Tor использует известные relay nodes — их IP известны и блокируются.
Мосты (bridges) помогают, но их тоже находят.
Скорость Tor: 1-5 Мбит/с в лучшем случае.

### Вывод

Любой инструмент, который **создаёт новый протокол** — рано или поздно детектируется.
Нужен инструмент, который **не создаёт ничего нового**, а использует то, что уже есть.

---

## 3. Ключевая идея

```
Нельзя заблокировать то, что неотличимо от того,
что ОБЯЗАТЕЛЬНО должно проходить всегда.
```

NEUTRINO не маскируется под легитимный трафик.
NEUTRINO **является** легитимным трафиком.

Данные передаются через:
1. DNS-запросы — обязательный протокол интернета
2. HTTPS к Cloudflare CDN — нельзя заблокировать без обрушения банков
3. NTP-пакеты — без них невалидны все SSL-сертификаты

Ни один из этих каналов не может быть заблокирован без уничтожения самого интернета в стране.

---

## 4. Архитектура

```
┌─────────────────────────────────────────────────────────────────┐
│                        КЛИЕНТ                                   │
│                                                                 │
│  Пользовательские данные                                        │
│         ↓                                                       │
│  [Encrypt: AES-256-GCM + X25519]                               │
│         ↓                                                       │
│  [Fragment: разбивка на чанки под каждый канал]                │
│         ↓                                                       │
│  ┌──────────────── MULTIPATH SCHEDULER ────────────────────┐   │
│  │                                                          │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌─────────────────┐ │   │
│  │  │  CDN/WS      │ │  DoH/DNS     │ │  NTP Timing     │ │   │
│  │  │  PRIMARY     │ │  SECONDARY   │ │  EMERGENCY      │ │   │
│  │  │  30-100Mbit  │ │  1-20 Mbit   │ │  ~0.1 Mbit      │ │   │
│  │  └──────┬───────┘ └──────┬───────┘ └────────┬────────┘ │   │
│  │         │                │                  │           │   │
│  └─────────┼────────────────┼──────────────────┼───────────┘   │
│            │                │                  │               │
└────────────┼────────────────┼──────────────────┼───────────────┘
             │                │                  │
             ▼                ▼                  ▼
     ┌───────────┐    ┌───────────┐    ┌──────────────┐
     │ Cloudflare│    │ 1.1.1.1   │    │ time.cloudflare│
     │ Workers   │    │ 8.8.8.8   │    │ pool.ntp.org  │
     │ (relay)   │    │ 9.9.9.9   │    │ (covert ch.)  │
     └─────┬─────┘    └─────┬─────┘    └──────┬───────┘
           │                │                  │
           └────────────────┴──────────────────┘
                            │
                            ▼
                   ┌─────────────────┐
                   │   RELAY NODE    │
                   │ (VPS / любой   │
                   │  сервер вне RU) │
                   └────────┬────────┘
                            │
                            ▼
                   ┌─────────────────┐
                   │   Интернет      │
                   └─────────────────┘
```

---

## 5. Канал 1 — DNS Covert Channel

### Принцип

DNS — Domain Name System. Без него не работает вообще ничего.
Каждый раз когда открываешь сайт — браузер делает DNS-запрос.

NEUTRINO прячет данные в **поддоменах** DNS-запросов и **TXT-записях** ответов.

### Кодирование данных

```
Входные данные: зашифрованный чанк байт

Кодирование: base32 (DNS case-insensitive, только [A-Z0-9-])

Запрос:
  a3f9b2c1d8e74f2a.b91c3d5e6f.seq0042.neutrino.example.com
  └──────────────────────────┘ └─────┘ └──────┘
        зашифрованные данные   checksum sequence_number

TXT ответ (до 65535 байт!):
  "v=n1 seq=42 data=BASE32ENCODEDRESPONSEDATA..."
```

### DNS over QUIC (DoQ) — RFC 9250

DoQ — новый стандарт, шифрует даже заголовки DNS.
QUIC = HTTP/3 транспорт, multiplexing встроен.
РКН его ещё не умеет блокировать.

```python
# Псевдокод клиента
import asyncio

async def dns_send_chunk(data: bytes, seq: int) -> bytes:
    encoded = base32_encode(data)
    subdomain = f"{encoded}.seq{seq:06d}.{RELAY_DOMAIN}"
    
    response = await doq_query(
        server="94.140.14.14",  # AdGuard DoQ
        name=subdomain,
        type="TXT"
    )
    return base32_decode(response.txt_record)
```

### Параллелизм для скорости

```python
# Одновременные запросы к разным резолверам
DOQ_RESOLVERS = [
    "quic://94.140.14.14:8853",   # AdGuard
    "quic://94.140.15.15:8853",   # AdGuard secondary
    "quic://1.1.1.1:8853",        # Cloudflare
    "quic://9.9.9.9:8853",        # Quad9
    "quic://dns.nextdns.io:8853", # NextDNS
]

# 10 параллельных запросов × 5 резолверов × 4096 байт TXT
# = 200 KB за раунд при latency 40ms
# = 5 MB/s = ~40 Мбит/с теоретически
# Реально с overhead: 1-5 Мбит/с
```

### Как выглядит для DPI

```
DPI видит:
  UDP port 853 (DNS over QUIC)
  Зашифрованный QUIC трафик к известным DNS-серверам (1.1.1.1, etc.)
  Частота: ~10-50 запросов/сек (нормально для браузера)

DPI думает:
  Пользователь активно серфит интернет, резолвит домены.

DPI не может:
  Заблокировать DNS без уничтожения всего интернета.
```

---

## 6. Канал 2 — CDN WebSocket Fronting

### Почему Cloudflare нельзя заблокировать

Cloudflare обслуживает:
- ~20% всего HTTP-трафика в мире
- Сбербанк, Тинькофф и другие российские банки
- Государственные сайты России
- Десятки тысяч российских компаний

Блокировка Cloudflare IP = экономический коллапс.
Это осознанное физическое ограничение для РКН.

### Cloudflare Workers как Relay

Cloudflare Workers — это JavaScript на CDN, исполняемый на ~300 edge-нодах планеты.

```javascript
// worker.js — деплоится на Cloudflare Workers (бесплатно!)
// Выглядит как обычный WebSocket сервис (чат, игра, что угодно)

export default {
  async fetch(request, env) {
    const upgradeHeader = request.headers.get("Upgrade");
    
    if (upgradeHeader !== "websocket") {
      // Для обычных HTTP запросов — возвращаем фейковую страницу
      return new Response("OK", { status: 200 });
    }

    // WebSocket соединение
    const [client, server] = Object.values(new WebSocketPair());
    
    server.accept();
    
    // Проксируем к реальному relay-серверу
    const relay = await connectToRelay(env.RELAY_URL, env.RELAY_KEY);
    
    server.addEventListener("message", ({ data }) => {
      relay.send(data);
    });
    
    relay.addEventListener("message", ({ data }) => {
      server.send(data);
    });

    return new Response(null, {
      status: 101,
      webSocket: client,
    });
  }
};
```

### Domain Fronting через SNI

```
HTTP запрос клиента:

  TLS ClientHello:
    SNI: cloudflare.com          ← видно РКН, выглядит легитимно
    
  Внутри TLS (зашифровано, РКН не видит):
    Host: neutrino-abc123.workers.dev  ← наш Worker
    Upgrade: websocket
    X-NEUTRINO-KEY: [HMAC auth]
```

Для DPI это HTTPS соединение к cloudflare.com.
Точка.

### TLS Fingerprint — uTLS

Проблема: разные клиенты имеют разный TLS fingerprint.
Браузер Chrome 120 имеет свой порядок extensions в ClientHello.
OpenVPN — свой. Go's tls package — свой.

Решение: **uTLS** (μTLS) — библиотека для Go/Rust, имитирующая точный fingerprint браузера.

```go
import utls "github.com/refraction-networking/utls"

config := &utls.Config{ServerName: "cloudflare.com"}
conn, _ := utls.Dial("tcp", "104.16.0.1:443", config)

// Имитируем Chrome 120 точно до байта
err = conn.ApplyPreset(&utls.HelloChrome_120)
```

Для DPI: это Chrome 120. Буквально.

---

## 7. Канал 3 — NTP Timing Steganography

### Зачем NTP

NTP (Network Time Protocol) — синхронизация часов.
Используется для валидации TLS-сертификатов.
Без NTP все HTTPS-сайты становятся недоступны.
Заблокировать нельзя.

### Timing Covert Channel

Идея: данные кодируются **не в содержимом пакетов**, а в **промежутках между ними**.
DPI анализирует содержимое — а данные в паузах.

```python
# Кодирование
def encode_bit_in_timing(bit: int) -> float:
    """
    bit=0 → задержка 100-110 μs (в пределах нормальных флуктуаций NTP)
    bit=1 → задержка 120-130 μs
    """
    if bit == 0:
        return 0.000100 + random.uniform(0, 0.000010)
    else:
        return 0.000120 + random.uniform(0, 0.000010)

def send_byte_via_ntp(byte_value: int, ntp_server: str):
    for bit_pos in range(8):
        bit = (byte_value >> bit_pos) & 1
        delay = encode_bit_in_timing(bit)
        
        send_ntp_request(ntp_server)
        time.sleep(delay)

# Декодирование
def decode_bit_from_timing(observed_delay: float) -> int:
    if 0.000095 <= observed_delay <= 0.000115:
        return 0
    elif 0.000115 < observed_delay <= 0.000135:
        return 1
    else:
        return -1  # шум, игнорировать
```

### Скорость и применение

```
Скорость: ~8 бит/сек при 1 NTP пакете/сек
         ~800 бит/сек при 100 пакетах/сек (ещё нормально)
         = ~100 байт/сек = ~0.8 Кбит/с

Применение: ТОЛЬКО для критических данных в аварийном режиме
  - Обмен ключами шифрования (X25519 = 32 байта = ~3 сек)
  - Bootstrap адреса для основных каналов
  - Короткие сообщения когда всё остальное заблокировано
```

Это последний рубеж. Не для видео — для выживания соединения.

---

## 8. Multipath Scheduler

### Задача

Автоматически выбирать и комбинировать каналы для достижения целевой скорости.
Без конфигурации со стороны пользователя.

### Алгоритм

```python
class MultipathScheduler:
    
    CHANNELS = {
        "cdn_ws":    {"priority": 1, "target_bw": 50_000_000},  # 50 Мбит
        "doh_http2": {"priority": 2, "target_bw": 10_000_000},  # 10 Мбит
        "dns_quic":  {"priority": 3, "target_bw": 2_000_000},   #  2 Мбит
        "dns_txt":   {"priority": 4, "target_bw": 500_000},     # 500 Кбит
        "ntp_timing":{"priority": 5, "target_bw": 800},         # 800 бит
    }
    
    def select_channels(self, required_bw: int) -> list[str]:
        """
        Выбираем минимальный набор каналов для required_bw
        Начинаем с самого быстрого, добавляем если не хватает
        """
        available = self.probe_all_channels()  # проверяем что работает
        selected = []
        current_bw = 0
        
        for ch_name, ch_config in sorted(
            self.CHANNELS.items(), 
            key=lambda x: x[1]["priority"]
        ):
            if ch_name in available and current_bw < required_bw:
                selected.append(ch_name)
                current_bw += available[ch_name]["measured_bw"]
        
        return selected
    
    def probe_channel(self, channel: str) -> dict:
        """
        Отправляем тестовый пакет, замеряем RTT и скорость
        Возвращаем None если канал заблокирован
        """
        try:
            start = time.monotonic_ns()
            response = self.send_probe(channel, probe_data=b"NEUTRINO_PROBE")
            rtt = (time.monotonic_ns() - start) / 1e6  # ms
            
            if response == b"NEUTRINO_PROBE_ACK":
                return {"rtt_ms": rtt, "measured_bw": self.measure_bw(channel)}
            return None
        except:
            return None  # канал недоступен
    
    def auto_failover(self):
        """
        Фоновая задача: каждые 30 сек перепроверяем каналы
        При деградации CDN — плавно переключаемся на DoH/DNS
        """
        while True:
            self.channels_state = {
                ch: self.probe_channel(ch) 
                for ch in self.CHANNELS
            }
            time.sleep(30)
```

### Сборка пакетов из нескольких каналов

```python
class PacketReassembler:
    """
    Пакеты летят через разные каналы в разном порядке.
    Собираем обратно по sequence numbers.
    """
    
    def send_data(self, data: bytes, channels: list[str]):
        chunks = self.split_into_chunks(data)
        
        for i, chunk in enumerate(chunks):
            # Добавляем sequence number и HMAC
            packet = NeutrinoPacket(
                seq=i,
                total=len(chunks),
                data=chunk,
                hmac=self.sign(chunk)
            )
            # Отправляем через следующий доступный канал (round-robin)
            channel = channels[i % len(channels)]
            self.send_via(channel, packet)
    
    def receive(self) -> bytes:
        buffer = {}
        while True:
            packet = self.receive_any_channel()
            if self.verify_hmac(packet):
                buffer[packet.seq] = packet.data
                if len(buffer) == packet.total:
                    return self.reassemble(buffer)
```

---

## 9. Шифрование

### Выбор алгоритмов

```
Key Exchange:   X25519 (Curve25519 ECDH)
  → Используется в Signal, WireGuard
  → 32 байта на публичный ключ
  → Perfect Forward Secrecy — новые ключи каждую сессию

Symmetric:      AES-256-GCM
  → AEAD — шифрование + аутентификация вместе
  → Аппаратное ускорение на всех современных CPU
  → 96-битный nonce, 128-битный tag

Key Derivation: HKDF-SHA256
  → Из shared secret X25519 выводим отдельные ключи
  → для каждого направления (client→server, server→client)

Identity:       Ed25519 signatures
  → Аутентификация сервера
  → Защита от MITM
```

### Handshake

```
CLIENT                                    SERVER

generate ephemeral keypair
(eph_priv, eph_pub)

→ ClientHello:
  neutrino_version: 1
  eph_pub: [32 bytes]
  timestamp: [8 bytes]  ← защита от replay
  
                              generate ephemeral keypair
                              (srv_eph_priv, srv_eph_pub)
                              
                              shared = X25519(srv_eph_priv, eph_pub)
                              keys = HKDF(shared, "neutrino-v1")
                              
                        ← ServerHello:
                            srv_eph_pub: [32 bytes]
                            signature: Ed25519(server_identity, srv_eph_pub)
                            
verify signature (Ed25519)
shared = X25519(eph_priv, srv_eph_pub)
keys = HKDF(shared, "neutrino-v1")

→ [все последующие данные зашифрованы AES-256-GCM]
```

### Packet Format

```
┌─────────────────────────────────────────────────────┐
│ NEUTRINO PACKET                                     │
├──────────┬──────────┬──────────┬───────────────────┤
│ Version  │ Seq Num  │ Channel  │ Encrypted Payload │
│ 1 byte   │ 4 bytes  │ 1 byte   │ N bytes           │
├──────────┴──────────┴──────────┤                   │
│ Nonce: 12 bytes                │ AES-256-GCM       │
│ Tag:   16 bytes                │ AEAD              │
└────────────────────────────────┴───────────────────┘

Total overhead per packet: 34 bytes
```

---

## 10. Как выглядит трафик для DPI

### Канал CDN/WebSocket

```
DPI наблюдает:
  Protocol:    TLS 1.3
  SNI:         cloudflare.com (или другой CDN домен)
  Destination: 104.16.0.0/12 (Cloudflare IP range)
  TLS Hello:   идентично Chrome 120 (uTLS)
  Pattern:     WebSocket соединение к CDN
  
DPI классифицирует:
  → Обычный пользователь, использует веб-приложение на Cloudflare
  → Всё нормально
  
DPI не может заблокировать:
  → Cloudflare IP используют тысячи российских сайтов
```

### Канал DNS/DoQ

```
DPI наблюдает:
  Protocol:    QUIC (UDP 853)
  Destination: 1.1.1.1, 8.8.8.8, 9.9.9.9 (известные DNS)
  Pattern:     Частые DNS запросы к стандартным резолверам
  
DPI классифицирует:
  → Пользователь активно серфит интернет
  → Нормальная активность браузера
  
DPI не может заблокировать:
  → 1.1.1.1 это Cloudflare DNS, используется миллионами
  → 8.8.8.8 это Google DNS, блокировка = Google недоступен
```

### Канал NTP

```
DPI наблюдает:
  Protocol:    UDP 123
  Destination: pool.ntp.org, time.cloudflare.com
  Pattern:     Периодические NTP пакеты (обычно 1/64 сек или реже)
  
DPI классифицирует:
  → Стандартная синхронизация времени ОС
  → Происходит на каждом компьютере
  
DPI не может заблокировать:
  → Блокировка NTP = невалидные TLS сертификаты везде
```

---

## 11. Таблица атак и защит

| Атака РКН | Инструмент | Почему не работает против NEUTRINO |
|---|---|---|
| Заблокировать IP сервера | Blacklist | IP сервера скрыт за CDN / меняется |
| Заблокировать домен | DNS blacklist | Домен Workers генерируется динамически ежедневно |
| DPI по TLS fingerprint | JA3/JA4 hash | uTLS имитирует Chrome 120 точно |
| DPI по паттерну трафика | ML classifier | Паттерн = обычный браузер на Cloudflare |
| Заблокировать Clo
