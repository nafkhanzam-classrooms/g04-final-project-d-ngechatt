[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/4SHtB1vz)

# NgeChat - Multi-Room Chat Application

NgeChat adalah aplikasi chat desktop multi-room berbasis TCP socket. Aplikasi ini
terdiri dari server Python yang menangani banyak koneksi client secara paralel
dan client GUI berbasis PyQt6 untuk login, membuat room, bergabung ke room,
chat publik, private message, kirim file, voice note, reaction, dan pengelolaan
friend list.

Komunikasi client-server berjalan di atas TLS lokal dengan payload JSON yang
dibungkus length-prefix framing. Data seperti user, room, membership, riwayat
pesan, attachment, reaction, dan friend list disimpan menggunakan SQLite.

## Link Penting

- Laporan:
[https://docs.google.com/document/d/1b-NyPvA9iN55qMu4jkGtlTZsI1OwZDYLu9ejwYA59ms](https://docs.google.com/document/d/1b-NyPvA9iN55qMu4jkGtlTZsI1OwZDYLu9ejwYA59ms/edit?usp=sharing)
- Video demo YouTube: [https://youtu.be/bgiH3aTaeR0](https://youtu.be/bgiH3aTaeR0)

## Fitur Utama

### Autentikasi dan sesi pengguna

- User dapat melakukan register, login, dan logout.
- Password disimpan dalam bentuk hash.
- Satu username hanya dapat login dari satu sesi aktif pada waktu yang sama.
- Daftar online user diperbarui ketika user login, logout, atau terputus.

### Multi-room chat

- User dapat membuat banyak room chat.
- Setiap room memiliki invite code yang dapat dibagikan ke user lain.
- User dapat join room melalui nama room dan invite code atau langsung melalui
  menu join by invite code.
- Pesan room dikirim sebagai broadcast ke semua client yang sedang aktif di room
  tersebut.
- Room owner dapat menghapus room, sedangkan member dapat keluar dari room.

### Private message dan friend system

- User dapat membuka private chat dengan user online atau friend.
- Riwayat private message tetap tersimpan sehingga percakapan dapat dibuka
  kembali.
- User dapat mengirim friend request, menerima/menolak request, melihat friend
  list, dan menghapus friend.
- Status online/offline pada friend list diperbarui dari server.

### Riwayat chat dan persistence

- Server menyimpan riwayat room chat dan private chat di SQLite.
- Saat user membuka room atau private chat, client mengambil 50 pesan terakhir.
- Pesan memiliki timestamp dan `message_id` agar attachment dan reaction dapat
  dikaitkan ke bubble chat yang benar.

### File, voice note, emoji, dan reaction

- User dapat mengirim file ke room atau private chat.
- File dibatasi sampai 5 MB untuk menjaga ukuran payload JSON tetap aman.
- File yang diterima dapat disimpan manual melalui tombol download di bubble chat.
- Voice note direkam dari microphone, dikirim sebagai file WAV, lalu dapat
  diputar langsung dari GUI.
- Emoji dapat disisipkan ke pesan.
- Reaction dapat diberikan pada pesan room atau private chat dan tersinkron ke
  penerima.

### Logging dan load testing

- Server menulis log ke console dan `logs/server.log`.
- `load_test.py` tersedia untuk simulasi banyak client, pengukuran latency, dan
  throughput pesan.

## Arsitektur Aplikasi

NgeChat memakai arsitektur client-server. Server menjadi pusat autentikasi,
membership room, routing pesan, dan persistence. Client GUI hanya menyimpan state
tampilan lokal, lalu semua aksi penting dikirim ke server sebagai packet JSON.

```text
PyQt6 GUI Client
  |
  | TLS over TCP
  | JSON packet + 4-byte length prefix
  v
Chat Server
  |-- ClientHandler thread per client
  |-- RoomManager untuk state online user dan active room
  |-- Database untuk SQLite persistence
  v
SQLite database + uploads/logs
```

### Server

Server dijalankan dari `server/server.py`. Komponen utamanya:

- `ChatServer`: membuka socket TCP, memasang TLS, menerima koneksi client, lalu
  membuat satu thread `ClientHandler` untuk setiap koneksi.
- `ClientHandler`: membaca packet dari satu client, memvalidasi tipe packet,
  menjalankan handler sesuai aksi, dan mengirim response/push packet.
- `RoomManager`: menyimpan state runtime seperti user online, socket aktif,
  room yang sedang aktif, dan lock pengiriman packet per socket.
- `Database`: membungkus akses SQLite secara thread-safe, membuat tabel, dan
  menyimpan user, room, membership, message, attachment, reaction, serta friend
  relationship.
- `server/protocol.py`: menyediakan helper framing, JSON serialization,
  validasi packet, dan builder packet response/push.

Model concurrency server adalah thread per client. Karena beberapa thread dapat
mengakses database dan socket pada saat yang sama, database dilindungi lock, SQLite
menggunakan WAL mode, dan pengiriman ke socket memakai lock per user agar packet
dari beberapa thread tidak saling bercampur.

### Client

Client GUI dijalankan dari `client/gui_main.py`. Komponen utamanya:

- `LoginWindow`: form koneksi, register, dan login.
- `MainWindow`: tampilan chat utama, daftar room, daftar online user, friend
  list, chat room, private chat, file transfer, voice note, dan reaction.
- `NetworkClient`: bridge networking yang berjalan di background thread. Packet
  dari server diteruskan ke GUI melalui Qt signal supaya update UI tetap aman.
- `client/protocol.py`: helper framing dan builder packet dari sisi client.

Client tidak melakukan broadcast sendiri. Semua pesan, file, voice note, dan
reaction dikirim ke server terlebih dahulu, lalu server melakukan routing ke room
atau user tujuan.

### Protokol komunikasi

Setiap packet memakai format:

```text
[4-byte big-endian payload length][UTF-8 JSON payload]
```

Contoh packet dari client:

```json
{"type":"login","username":"alice","password":"secret"}
{"type":"create_room","room":"Progjar"}
{"type":"join_by_code","code":"ABCD1234"}
{"type":"broadcast","room":"Progjar","message":"Halo!"}
{"type":"private_message","target":"bob","message":"Ping"}
{"type":"file_transfer","scope":"room","room":"Progjar","filename":"demo.txt","data":"...base64...","kind":"file"}
{"type":"reaction","scope":"room","room":"Progjar","message_id":"room-...","emoji":"like"}
```

Contoh packet dari server:

```json
{"status":"ok","message":"Login successful."}
{"type":"room_list","rooms":[{"room_name":"Progjar","created_by":"alice","invite_code":"","is_member":false}]}
{"type":"user_list","users":["alice","bob"]}
{"type":"broadcast","room":"Progjar","sender":"alice","message":"Halo!","timestamp":"2026-06-13 10:00:00 UTC","message_id":"room-..."}
{"type":"history","room":"Progjar","messages":[]}
```

## Struktur Project

```text
server/
  server.py        # TCP/TLS server, ClientHandler, packet dispatch
  protocol.py      # JSON framing, serialization, validasi packet
  room_manager.py  # online users, active room membership, push delivery
  database.py      # SQLite persistence
  logger.py        # console/file logging
client/
  gui_main.py      # GUI entry point
  network_client.py
  protocol.py
  gui/
    login_window.py
    main_window.py
    dialogs.py
    styles.py
certs/
  cert.pem         # self-signed certificate untuk TLS lokal
  key.pem
database/
  chat.db          # SQLite database lokal
downloads/         # file hasil download dari client
uploads/           # file/voice note yang diterima server
logs/
  server.log
load_test.py
requirements.txt
```

## Setup

Gunakan Python 3.10+.

```bash
pip install -r requirements.txt
```

`PyAudio` digunakan untuk merekam voice note dari microphone. Jika instalasi
PyAudio bermasalah di Windows, gunakan versi Python yang memiliki wheel PyAudio
yang sesuai atau install dependency PortAudio terlebih dahulu.

File TLS sudah tersedia di folder `certs/`. Jika `certs/cert.pem` atau
`certs/key.pem` tidak ada, buat self-signed certificate baru, misalnya dengan
OpenSSL:

```bash
mkdir certs
openssl req -x509 -newkey rsa:2048 -nodes -keyout certs/key.pem -out certs/cert.pem -days 365
```

## Menjalankan Aplikasi

Jalankan server dari terminal pertama:

```bash
python -m server.server
```

Mode debug dapat dinyalakan jika ingin log lebih detail:

```bash
python -m server.server --debug
```

Jalankan client GUI dari terminal kedua dan terminal berikutnya:

```bash
python -m client.gui_main
```

Default server adalah `127.0.0.1:9090` dari sisi client dan `0.0.0.0:9090` dari
sisi server. Host dan port bisa diubah dari login form atau environment variable:

```bash
set CHAT_HOST=127.0.0.1
set CHAT_PORT=9090
```

Di PowerShell:

```powershell
$env:CHAT_HOST = "127.0.0.1"
$env:CHAT_PORT = "9090"
```

## Cara Pakai Singkat

1. Jalankan server, lalu buka dua atau lebih client GUI.
2. Register akun baru, kemudian login.
3. Buat room melalui tombol `Add Room`, lalu simpan atau bagikan invite code.
4. Client lain dapat join memakai invite code.
5. Pilih room untuk mengirim broadcast message.
6. Pilih user online atau friend untuk membuka private chat.
7. Gunakan tombol `File`, `Voice`, dan emoji sesuai kebutuhan.
8. Klik kanan bubble chat untuk memberi atau menghapus reaction.
9. Gunakan menu friend untuk add friend, menerima request, atau remove friend.

## Pengujian Beban

Setup room untuk load test:

```bash
python load_test.py --setup
```

Catat invite code yang muncul, lalu jalankan simulasi:

```bash
python load_test.py --clients 20 --messages 50 --code KODE_ROOM
```

Output load test berisi jumlah client yang berhasil connect, pesan berhasil dan
gagal, latency minimum/rata-rata/maksimum, throughput pesan per detik, serta file
hasil `load_test_result_YYYYMMDD_HHMMSS.txt`.

## Catatan Implementasi

- TLS menggunakan self-signed certificate untuk kebutuhan demo lokal.
- Client menerima self-signed certificate dengan `ssl.CERT_NONE`, sehingga setup
  lokal tidak perlu certificate authority.
- SQLite memakai WAL mode dan lock Python agar aman dipakai oleh banyak
  `ClientHandler` thread.
- Attachment disimpan sebagai metadata di database dan file fisik di `uploads/`.
- Payload file dikirim sebagai base64 di dalam JSON, sehingga ukuran transfer
  dibatasi 5 MB.

## Anggota Kelompok

| Name | NRP |
|------|-----|
| Hisyam Syafa Raditya | 5025241130 |
| A. Wildan Kevin Assyauqi | 5025241265 |
