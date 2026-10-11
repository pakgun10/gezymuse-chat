# Implementation Plan — GezyChat

Aplikasi percakapan real-time ala WhatsApp/Telegram untuk lingkungan sekolah
(guru, TU, wali kelas, MGMP). Dokumen ini adalah rencana lengkap dari nol
sampai MVP.

**Stack yang ditetapkan:** Bun + Hono + WebSocket native + MariaDB + Redis.
**UI:** `index.html` (prototipe yang sudah ada — tampilan final, tinggal
disambungkan ke backend).

---

## 1. Visi & Target MVP

**Visi:** satu aplikasi chat milik sendiri — data di server sendiri (VPS),
tanpa ketergantungan ke platform asing, cukup untuk kebutuhan komunikasi
sekolah.

**Target MVP (selesai = Bagian 10 terpenuhi):**

- Login dengan nomor HP + kode OTP (ala WhatsApp).
- Chat 1-on-1 dan grup kecil (maks ~50 anggota).
- Pesan teks, gambar, dokumen, voice note.
- Status pesan: terkirim (✓) → diterima (✓✓) → dibaca (✓✓ biru).
- Indikator "mengetik..." dan status online.
- Riwayat pesan dengan pagination ("muat pesan lama").
- Push notification saat aplikasi di-background (FCM).
- Antrian offline: pesan terkirim otomatis saat sinyal kembali.
- Berjalan di HP via PWA dan/atau APK bungkus WebView (pola Bimasoft).

**Asumsi kapasitas MVP:**

- 1.000 akun terdaftar.
- Hingga 1.000 koneksi WebSocket aktif secara bersamaan.
- Sekitar 30 pengguna aktif mengirim pesan pada periode sibuk.
- Sistem dirancang untuk puncak awal 30 pesan masuk per detik dan fan-out
  grup sampai 50 anggota.
- Target latency pesan p95 di bawah 1 detik pada kondisi jaringan normal.
- Satu instance Bun pada VPS cukup untuk target ini; scale-out dilakukan
  setelah metrik nyata menunjukkan kebutuhan.

**Bukan target MVP (ditunda — Bagian 12):** enkripsi end-to-end, panggilan
suara/video, channel/broadcast besar, sinkron multi-device penuh.

MVP dapat mendaftarkan beberapa perangkat untuk push notification. Status
chat dan cursor dibaca tetap dihitung per pengguna; sinkronisasi penuh antar
perangkat ditunda.

---

## 2. Arsitektur

```
┌─────────────────────┐      HTTPS / WSS      ┌─────────┐
│  PWA / APK WebView  │ ◄──────────────────► │  nginx  │
│  (index.html + JS)  │                       │  (TLS)  │
└─────────────────────┘                       └────┬────┘
                                                   │
                                            ┌──────▼──────┐
                                            │ Bun + Hono  │
                                            │  • REST API │
                                            │  • WS /ws   │
                                            └─┬─┬─┬─┬─────┘
                                              │ │ │ └─► Storage media
                                              │ │ │     (disk VPS / R2)
                                              │ │ └─► Redis
                                              │ │     (pub/sub, presence,
                                              │ │      rate-limit, OTP)
                                              │ └─► MariaDB
                                              │     (users, percakapan,
                                              │      pesan)
                                              └─► FCM (push notification)
```

- Satu binary Bun melayani REST + WebSocket sekaligus.
- Untuk target MVP, satu instance Bun melayani trafik aplikasi. Redis tetap
  dipakai untuk OTP, presence, dan rate limit.
- Jika nanti perlu scale-out, instance Bun tambahan ditempatkan di belakang
  nginx. Fan-out antar-instance memakai Redis Pub/Sub, sedangkan MariaDB
  tetap menjadi sumber kebenaran dan reconnect mengambil data dari database.

---

## 3. Stack Detail

| Lapisan      | Pilihan | Catatan |
|--------------|---------|---------|
| Runtime      | **Bun** | Sudah dikuasai; WebSocket native cepat |
| Framework    | **Hono** | Router REST; jalan di atas `Bun.serve` |
| Realtime     | **WebSocket native Bun** (`Bun.serve({ websocket })`) | Endpoint `/ws`; Hono untuk upgrade handshake |
| Database     | **MariaDB 10.11+** | Driver `mysql2/promise` atau Drizzle ORM |
| Cache/bus    | **Redis 7** | Pub/sub, presence, rate-limit, penyimpanan OTP |
| Frontend     | `index.html` prototipe (vanilla JS) | Nanti: Tailwind; tetap satu file |
| PWA          | manifest + service worker | Fase 4; IndexedDB untuk antrian offline |
| Android      | APK bungkus WebView | Push native memerlukan Firebase Messaging dan bridge ke JavaScript |
| Push         | FCM HTTP v1/Admin SDK + Web Push | Android dan PWA memakai jalur konfigurasi yang berbeda |
| Media        | Object storage atau disk privat VPS | URL akses sementara; alternatif: Cloudflare R2 (S3-compatible) |
| Auth         | OTP via SMS/WA gateway → JWT (HS256) | Access token 15 mnt + refresh token yang dapat dicabut |
| Deploy       | VPS, systemd + nginx + certbot | Pola yang sama dengan GezyForm |
| Observability | Pino + endpoint health + metrik dasar | Koneksi WS, latency, error, DB, Redis, disk |

**Keputusan sadar:** tidak memakai Socket.IO (butuh kompatibilitas
fallback polling — tidak perlu), tidak memakai protokol biner custom
(MTProto dsb. — kelas raksasa, JSON cukup).

---

## 4. Skema Database (MariaDB)

```sql
CREATE TABLE users (
  id            BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  phone         VARCHAR(20)  NOT NULL,              -- +62812xxxx, format E.164
  name          VARCHAR(100) NOT NULL,
  role          ENUM('admin','guru','tu','wali_kelas','mgmp','siswa')
                NOT NULL DEFAULT 'siswa',
  avatar_url    VARCHAR(255) NULL,
  created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uq_users_phone (phone)
);

CREATE TABLE conversations (
  id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  type       ENUM('direct','group') NOT NULL,
  title      VARCHAR(120) NULL,                    -- nama grup; NULL utk direct
  created_by BIGINT UNSIGNED NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (created_by) REFERENCES users(id) ON DELETE SET NULL
);

CREATE TABLE direct_conversation_pairs (
  conversation_id BIGINT UNSIGNED NOT NULL,
  user_a_id       BIGINT UNSIGNED NOT NULL,
  user_b_id       BIGINT UNSIGNED NOT NULL,
  PRIMARY KEY (conversation_id),
  UNIQUE KEY uq_direct_pair (user_a_id, user_b_id),
  FOREIGN KEY (conversation_id) REFERENCES conversations(id) ON DELETE CASCADE,
  FOREIGN KEY (user_a_id) REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (user_b_id) REFERENCES users(id) ON DELETE CASCADE,
  CHECK (user_a_id < user_b_id)
);

CREATE TABLE conversation_members (
  conversation_id   BIGINT UNSIGNED NOT NULL,
  user_id           BIGINT UNSIGNED NOT NULL,
  role              ENUM('admin','member') DEFAULT 'member',
  last_delivered_msg_id BIGINT UNSIGNED NULL,        -- cursor delivered per user
  last_read_msg_id  BIGINT UNSIGNED NULL,          -- untuk badge & "dibaca"
  joined_at         TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (conversation_id, user_id),
  FOREIGN KEY (conversation_id) REFERENCES conversations(id) ON DELETE CASCADE,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  INDEX idx_member_user (user_id, conversation_id)
);

CREATE TABLE messages (
  id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  conversation_id BIGINT UNSIGNED NOT NULL,
  sender_id       BIGINT UNSIGNED NOT NULL,
  client_message_id VARCHAR(64) NOT NULL,            -- stabil saat retry/reconnect
  kind            ENUM('text','image','file','audio') NOT NULL,
  body_text       TEXT NULL,                        -- teks / caption
  media_key       VARCHAR(255) NULL,                 -- object key; bukan URL publik
  media_mime      VARCHAR(100) NULL,
  media_size      INT UNSIGNED NULL,
  reply_to_id     BIGINT UNSIGNED NULL,
  created_at      TIMESTAMP(3) DEFAULT CURRENT_TIMESTAMP(3),
  FOREIGN KEY (conversation_id) REFERENCES conversations(id) ON DELETE CASCADE,
  FOREIGN KEY (sender_id) REFERENCES users(id),
  FOREIGN KEY (reply_to_id) REFERENCES messages(id) ON DELETE SET NULL,
  UNIQUE KEY uq_msg_sender_client (sender_id, client_message_id),
  INDEX idx_conv_time (conversation_id, id),        -- pagination cepat
  INDEX idx_msg_sender (sender_id, id)
);

CREATE TABLE devices (                              -- untuk push
  user_id    BIGINT UNSIGNED NOT NULL,
  device_id  VARCHAR(128) NOT NULL,
  fcm_token  VARCHAR(255) NOT NULL,
  platform   ENUM('android','web') NOT NULL,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (user_id, device_id),
  UNIQUE KEY uq_devices_fcm_token (fcm_token),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE TABLE refresh_tokens (
  id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  user_id    BIGINT UNSIGNED NOT NULL,
  device_id  VARCHAR(128) NULL,
  token_hash CHAR(64) NOT NULL,
  expires_at DATETIME NOT NULL,
  revoked_at DATETIME NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uq_refresh_token_hash (token_hash),
  INDEX idx_refresh_user (user_id, revoked_at, expires_at),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

OTP **tidak** disimpan di MariaDB — cukup di Redis dengan TTL 5 menit
(bagian 8). Access token JWT tetap stateless, tetapi refresh token disimpan
sebagai hash agar dapat dicabut per perangkat.

`last_delivered_msg_id` dan `last_read_msg_id` adalah cursor per anggota.
Server hanya boleh memajukan cursor, dan harus memastikan pesan tersebut
berasal dari percakapan yang sama. Batas maksimal 50 anggota grup, pasangan
direct yang unik, serta kecocokan `kind` dengan `body_text`/`media_key`
divalidasi dalam transaksi aplikasi.

---

## 5. Protokol WebSocket (JSON)

Satu koneksi per perangkat ke `wss://<host>/ws`. Token dikirim pada event
`auth`, bukan pada query string. Envelope selalu:

```jsonc
{ "t": "<event>", "d": { /* payload */ }, "reqId": "uuid-opsional" }
```

`reqId` hanya berada di envelope dan harus tetap sama ketika client mengulang
permintaan. Server memvalidasi ukuran frame, bentuk payload, dan event yang
diizinkan setelah autentikasi.

**Client → Server**

| Event | Payload | Fungsi |
|-------|---------|--------|
| `auth` | `{ "token": "<JWT>" }` | Wajib pertama; tanpa ini koneksi ditutup |
| `msg.send` | `{ "convId": 12, "clientMessageId": "uuid", "kind": "text", "body": "halo", "replyTo": null }` | Kirim pesan; `reqId` envelope untuk ack/idempotency |
| `msg.read` | `{ "convId": 12, "lastMsgId": 345 }` | Tandai dibaca → update `last_read_msg_id` |
| `typing` | `{ "convId": 12, "state": true }` | Mulai/berhenti mengetik |
| `sync` | `{ "cursors": [{ "convId": 12, "lastMsgId": 345 }] }` | Ambil pesan yang terlewat setelah reconnect |
| `ping` | `{}` | Heartbeat tiap 30 dtk (jawaban `pong`) |

**Server → Client**

| Event | Payload | Fungsi |
|-------|---------|--------|
| `msg.new` | `{ "msg": { id, convId, senderId, kind, body, mediaUrl, createdAt } }` | Pesan masuk → render bubble |
| `msg.ack` | `{ "reqId": "abc", "msgId": 346, "status": "sent" }` | Optimistic UI: ✓ satu |
| `msg.status` | `{ "msgId": 346, "convId": 12, "status": "delivered"/"read", "byUserId": 7 }` | ✓✓ / ✓✓ biru |
| `sync.done` | `{ "messages": [], "cursors": [] }` | Menandai sinkronisasi reconnect selesai |
| `typing` | `{ "convId": 12, "userId": 7, "name": "Bu Sari", "state": true }` | Indikator mengetik |
| `presence` | `{ "userId": 7, "online": true }` | Status online/offline |
| `error` | `{ "reqId": "abc", "code": "FORBIDDEN", "message": "..." }` | Error terstruktur untuk client |
| `pong` | `{}` | Jawaban heartbeat |

**Aturan penting:**

- Server selalu simpan ke MariaDB **dulu**, baru fan-out. Tidak ada pesan
  yang "hanya di memori".
- `msg.ack` dikirim setelah transaksi MariaDB berhasil commit. `clientMessageId`
  harus unik per pengirim; retry dengan ID yang sama mengembalikan pesan lama
  tanpa membuat pesan baru.
- `delivered` = pesan sudah sampai ke WS penerima yang online. Jika penerima
  offline → tetap `sent`, lalu server mengirim push FCM. Cursor delivered
  diperbarui ketika event berhasil diteruskan ke koneksi penerima.
- `read` hanya ditetapkan setelah client mengirim `msg.read`; server tidak
  boleh menganggap pesan terbaca hanya karena pesan sudah delivered.
- Setelah reconnect dan `auth`, client mengirim `sync` dengan `lastMsgId` per
  percakapan. Server mengambil data dari MariaDB dan mengirim pesan yang
  terlewat, sehingga Redis atau WebSocket yang sempat putus tidak menyebabkan
  pesan hilang.
- Semua event memeriksa autentikasi dan keanggotaan percakapan. Client tidak
  boleh menentukan `senderId`, status pesan, atau URL media.
- Heartbeat server dan client berjalan setiap 30 detik; koneksi tanpa auth
  dalam 5 detik dan koneksi yang tidak merespons heartbeat ditutup.

---

## 6. REST API (Hono)

Semua di bawah `/api`, auth via `Authorization: Bearer <JWT>`
(kecuali dua endpoint OTP pertama).

```
POST /api/auth/otp/request   { phone }            → { ok, retryAfter }
POST /api/auth/otp/verify    { phone, code }      → { accessToken, refreshToken, user }
POST /api/auth/refresh       { refreshToken, deviceId? } → { accessToken, refreshToken? }
POST /api/auth/logout        { refreshToken? }    → { ok }

GET  /api/conversations                          → daftar chat + last message + unread count
POST /api/conversations      { type, title?, memberPhones[] }
POST /api/conversations/:id/members { userId }    → tambah anggota grup
GET  /api/conversations/:id/messages?before=<msgId>&limit=30
POST /api/media/upload       multipart(form-data) → { url, mime, size }
GET  /api/users/me
PUT  /api/users/me           { name, avatar? }
POST /api/devices            { deviceId, fcmToken, platform }
```

Upload dibatasi: gambar ≤ 5 MB, dokumen ≤ 20 MB, audio ≤ 10 MB; validasi
MIME berdasarkan isi file di server; nama file diacak (UUID) agar tidak bisa
ditebak. File disimpan di luar direktori publik. `media_url` yang dikirim ke
client adalah URL sementara atau endpoint yang memeriksa keanggotaan
percakapan, bukan URL file permanen.

Semua endpoint selain request/verify OTP memeriksa JWT, hak akses pengguna,
dan batas input. Pagination membatasi `limit` maksimum 100. Rate limit
diterapkan berdasarkan nomor HP, alamat IP, pengguna, dan endpoint untuk
mencegah penyalahgunaan OTP, pesan, dan upload.

---

## 7. Struktur Proyek

```
gezymuse-chat/
├── index.html               # prototipe UI (frontend v0) — JANGAN diubah tampilannya
├── implementation_plan.md   # dokumen ini
├── server/
│   ├── package.json         # bun init; deps: hono, mysql2, ioredis
│   ├── .env.example         # DB_URL, REDIS_URL, JWT_SECRET, FCM_*
│   ├── src/
│   │   ├── index.ts         # Bun.serve + app Hono + upgrade /ws
│   │   ├── health.ts         # readiness/liveness dan pemeriksaan dependensi
│   │   ├── validation.ts     # validasi payload REST dan WebSocket
│   │   ├── ws/
│   │   │   ├── manager.ts   # userId → Set<WebSocket>; auth; heartbeat
│   │   │   └── handler.ts   # switch event t: msg.send, typing, ...
│   │   ├── routes/
│   │   │   ├── auth.ts      # OTP + JWT
│   │   │   ├── conversations.ts
│   │   │   ├── messages.ts
│   │   │   └── media.ts     # upload
│   │   ├── db/
│   │   │   ├── migrations/   # migrasi berurutan, bukan schema manual saja
│   │   │   │   └── 001_initial.sql
│   │   │   └── index.ts     # pool mysql2
│   │   ├── redis.ts         # client ioredis + helper pub/sub
│   │   ├── storage.ts        # media privat dan signed URL
│   │   ├── fcm.ts            # push Android; memakai FCM HTTP v1/Admin SDK
│   │   └── metrics.ts        # metrik dasar dan request/correlation ID
│   └── Dockerfile           # opsional; pola utama tetap systemd (bagian 11)
└── pwa/                     # fase 4: manifest.json, sw.js, ikon, IndexedDB
```

**Perubahan di `index.html` saat disambungkan ke backend** (tampilan tetap):

1. Ganti fungsi `kirimPesan()` dummy → `ws.send({t:'msg.send', ...})` dengan
   `clientMessageId` yang disimpan di IndexedDB.
2. Ganti `setTimeout` balasan dummy → `ws.onmessage` → render bubble.
3. Tambah `Authorization` saat fetch riwayat + `auth` pertama saat WS connect.
4. Indikator mengetik & presence dikirim/diterima via event WS (bagian 5).
5. Saat reconnect, kirim cursor `lastMsgId` melalui event `sync` dan retry
   pesan berstatus `pending`/`sending` dengan ID client yang sama.

---

## 8. Redis — Untuk Apa Saja

| Key / Channel | Tipe | TTL | Fungsi |
|---------------|------|-----|--------|
| `otp:<phone>` | string | 300 dtk | Kode OTP 6 digit |
| `presence:<userId>` | string `"1"` | 60 dtk | Online; di-refresh heartbeat 30 dtk |
| `ratelimit:msg:<userId>` | counter | 60 dtk | Maks 60 pesan/menit/user |
| `ratelimit:otp:<phone>` | counter | 600 dtk | Maks 5 OTP/10 mnt/nomor |
| `ratelimit:otp:ip:<ip>` | counter | 600 dtk | Batas OTP per alamat IP |
| `chat:events` | pub/sub | — | Event lintas instance untuk `msg.new`, typing, status |
| `chat:user:<userId>` | pub/sub | — | Event personal jika routing khusus diperlukan |

Pada satu instance, fan-out dilakukan langsung ke socket lokal. Saat
scale-out, setiap instance cukup subscribe ke channel bersama `chat:events`
dan meneruskan event ke socket lokal yang relevan; jangan membuat satu
subscription Redis untuk setiap percakapan aktif.

Redis Pub/Sub bersifat sementara dan bukan antrean durable. MariaDB tetap
menjadi sumber kebenaran. Jika event lintas instance terlewat, event tersebut
akan ditemukan melalui `sync` saat reconnect. Jika kebutuhan delivery lintas
instance meningkat, tambahkan outbox table/worker atau Redis Streams.

---

## 9. Tahapan Implementasi

### Fase 0 — Fondasi (±1–2 hari)
- [ ] `server/` Bun + Hono jalan; `GET /` menyajikan `index.html`.
- [ ] Migrasi database awal (bagian 4) dijalankan; koneksi pool MariaDB ok.
- [ ] Koneksi Redis ok; `docker-compose.yml` lokal (mariadb + redis) untuk dev.
- [ ] `.env.example`; README cara jalan lokal.
- [ ] Endpoint health/readiness dan structured logging dengan correlation ID.
- [ ] Konfigurasi batas ukuran request, frame WebSocket, dan koneksi database.

### Fase 1 — Auth (±2–3 hari)
- [ ] `POST /api/auth/otp/request` + `verify` (OTP dummy `123456` di dev,
      SMS/WA gateway di produksi).
- [ ] OTP tidak membocorkan apakah nomor terdaftar; rate limit per nomor dan IP.
- [ ] JWT access (15 mnt) + refresh (30 hari) yang tersimpan sebagai hash dan
      dapat dicabut per perangkat.
- [ ] Otorisasi dasar dan validasi input untuk semua endpoint yang sudah ada.
- [ ] WS `/ws` menolak koneksi tanpa `auth` dalam 5 detik dan memeriksa origin.

### Fase 2 — Inti chat 1-on-1 (±1 minggu) ← **jantung MVP**
- [ ] `msg.send` → transaksi DB → `msg.ack` → fan-out `msg.new`.
- [ ] Idempotency dengan `clientMessageId`; retry tidak membuat pesan ganda.
- [ ] `msg.status`: delivered (sampai di WS) & read (via `msg.read`) memakai
      cursor per anggota.
- [ ] `GET /messages` pagination; daftar percakapan + unread count.
- [ ] Event `sync` setelah reconnect berdasarkan `lastMsgId` per percakapan.
- [ ] Frontend `index.html` disambungkan (5 perubahan di bagian 7).

### Fase 3 — Grup + kehadiran (±4–5 hari)
- [ ] Grup: buat, tambah anggota, `conversation_members`, maksimum 50 anggota.
- [ ] Direct conversation memakai pasangan pengguna unik agar tidak duplikat.
- [ ] `typing` + `presence` (heartbeat Redis).
- [ ] Reply (`reply_to_id`) — UI-nya sudah ada di prototipe.

### Fase 4 — Media + push + PWA (±1 minggu)
- [ ] Upload gambar/dokumen/voice note (bagian 6); render di bubble dengan
      URL sementara dan pemeriksaan akses.
- [ ] FCM HTTP v1/Admin SDK untuk Android dan Web Push/VAPID untuk PWA.
- [ ] APK WebView memakai Firebase Messaging native dan bridge jika push
      harus tetap berjalan saat WebView di-background.
- [ ] `manifest.json` + `sw.js`: instalasi + antrian offline
      (pesan gagal kirim tersimpan di IndexedDB, terkirim otomatis).

### Fase 5 — Hardening & deploy (±3–4 hari)
- [ ] Rate limit lanjutan, validasi semua input (zod), CSP, dan sanitasi output
      pesan untuk mencegah XSS.
- [ ] Uji 2 HP Android nyata: kirim, offline→online, background→push.
- [ ] Uji 1.000 koneksi WebSocket, 30 pengirim aktif, puncak 30 pesan/detik,
      dan grup 50 anggota.
- [ ] Uji retry, reconnect serentak, pesan ganda, akses percakapan ilegal,
      akses media ilegal, serta pencabutan refresh token.
- [ ] Deploy VPS: systemd unit, nginx reverse proxy + certbot (WSS),
      MariaDB & Redis di VPS yang sama.
- [ ] Backup database dan media; uji restore; metrik dasar dan alert disk/DB.
- [ ] 0 console error di frontend; log server rapi (pino).

**Estimasi total: 4–6 minggu** (paruh waktu, satu orang).

---

## 10. Definition of Done — MVP Dinyatakan Selesai Jika:

- [ ] Dua akun (2 nomor HP) bisa OTP-login dan saling chat teks < 1 detik (WiFi).
- [ ] Status pesan bergerak penuh: ✓ → ✓✓ → ✓✓ biru di kedua sisi.
- [ ] Grup 50 orang: anggota yang berhak menerima pesan + indikator mengetik
      muncul tanpa duplikasi.
- [ ] Kirim foto, dokumen PDF, dan voice note bisa dibuka di penerima.
- [ ] Pesan dikirim saat offline → terkirim otomatis saat online (tanpa kirim ulang manual).
- [ ] Penerima yang aplikasinya di-background menerima push FCM.
- [ ] Riwayat 500 pesan: scroll + "muat lama" lancar di HP kentang.
- [ ] Tidak ada pesan hilang/ganda setelah putus-sambung 10x (uji reconnect).
- [ ] Retry request dengan `clientMessageId` yang sama tidak membuat pesan ganda.
- [ ] Pengguna tidak dapat membaca atau mengirim pesan ke percakapan yang
      bukan anggotanya.
- [ ] URL media tanpa hak akses tidak dapat dibuka.
- [ ] 1.000 koneksi WebSocket dan 30 pengirim aktif memenuhi target p95
      latency < 1 detik pada uji beban.
- [ ] Refresh token dapat dicabut; OTP dan upload menolak percobaan berlebihan.
- [ ] Backup database/media berhasil dipulihkan pada lingkungan uji.
- [ ] Berjalan 24 jam di VPS tanpa restart darurat.

---

## 11. Deploy (VPS — pola GezyForm)

```
# /etc/systemd/system/gezychat.service
[Unit]
Description=GezyChat server
After=network-online.target
Wants=network-online.target

[Service]
WorkingDirectory=/opt/gezychat/server
EnvironmentFile=/opt/gezychat/server/.env
User=gezychat
Group=gezychat
ExecStart=/usr/local/bin/bun run src/index.ts
Restart=always
RestartSec=3
NoNewPrivileges=true
PrivateTmp=true
LimitNOFILE=8192

[Install]
WantedBy=multi-user.target
```

- nginx: `location /` → proxy ke `127.0.0.1:3024`; `location /ws` → proxy
  dengan header `Upgrade`/`Connection`, `proxy_read_timeout` yang cukup untuk
  WS, dan `client_max_body_size` sekitar 25 MB; TLS via certbot.
- Jalankan MariaDB dan Redis sebagai service terpisah; port database hanya
  listen di localhost/private network dan tidak dibuka ke internet.
- Ukuran awal: 2 vCPU, RAM 4 GB, SSD 40–80 GB. Naikkan kapasitas setelah
  metrik CPU, RAM, koneksi DB, latency, dan disk menunjukkan kebutuhan.
- Kredensial FCM server disimpan sebagai secret di server. `google-services.json`
  hanya untuk aplikasi Android, bukan kredensial pengiriman dari backend.
- Backup: `mariadb-dump` harian dan backup media ke lokasi terpisah; lakukan
  uji restore berkala, bukan hanya membuat file backup.

---

## 12. Ditunda Sampai Setelah MVP

1. **E2EE** (Signal Protocol / libsignal) — butuh manajemen kunci per
   perangkat; proyek tersendiri.
2. Panggilan suara/video (WebRTC + SFU seperti LiveKit).
3. Multi-device penuh (satu akun di 2 HP sinkron).
4. Channel/broadcast & grup > 500 anggota.
5. Enkripsi media at-rest dengan KMS; retensi & hapus otomatis.
6. Versi iOS native / desktop (Electron/Tauri).

---

## 13. Risiko & Mitigasi

| Risiko | Mitigasi |
|--------|----------|
| WS putus-putus di jaringan seluler | Heartbeat 30 dtk + `sync` via `lastMsgId` (bagian 5) |
| Redis Pub/Sub kehilangan event sementara | MariaDB sebagai sumber kebenaran; reconnect melakukan `sync`; outbox/Streams saat scale-out besar |
| Push Android/WebView tidak konsisten | FCM native untuk APK, Web Push/VAPID untuk PWA; uji perangkat nyata sejak Fase 4 |
| FCM butuh kredensial server dan konfigurasi client | FCM HTTP v1/Admin SDK di backend; `google-services.json` hanya di client Android |
| Spam/flood OTP, pesan, atau upload | Rate limit Redis per IP/nomor/user/endpoint sejak Fase 1 |
| Pesan ganda saat retry | `clientMessageId` + unique key `(sender_id, client_message_id)` |
| Akses pesan/media oleh pengguna yang tidak berhak | Otorisasi pada setiap REST/WS event dan signed URL/media endpoint |
| Nomor HP murid/guru berubah | Login = OTP ulang; `phone` unique, id internal stabil |
| Beban DB saat grup ramai | Index `(conversation_id, id)` dan `(user_id, conversation_id)`; pagination, bukan load semua |
| Backup ada tetapi tidak dapat dipulihkan | Uji restore database dan media secara berkala |

---

*Terakhir diperbarui: 2026-10-11. UI prototipe: `index.html` (terverifikasi
0 console error, responsif HP).*
