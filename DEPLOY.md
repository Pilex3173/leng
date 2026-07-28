# Panduan Deploy — Truf 108 Online

Arsitektur: **backend (server wasit)** jalan di VPS Anda, **frontend (tampilan)**
di-deploy statis ke Vercel. Keduanya komunikasi lewat WebSocket (Socket.IO).

```
Browser Teman 1 ─┐
Browser Teman 2 ─┼─► wss://truf-api.domainanda.com  (Node.js di VPS Anda)
Browser Teman 3 ─┘        ▲
                           │ frontend statis di-serve dari
                  https://truf-anda.vercel.app  (Vercel)
```

---

## 1. Backend — di VPS Anda

### 1.1 Upload & pasang dependency

```bash
# di VPS
mkdir -p ~/apps/truf-server && cd ~/apps/truf-server
# upload isi folder server/ (server.js, package.json) ke sini, lalu:
npm install --production
```

### 1.2 Pilih port

Default di `server.js` adalah **8181**. Cek dulu port itu bentrok atau tidak
dengan node blockchain yang sudah jalan di VPS Anda:

```bash
sudo ss -tulpn | grep 8181
```

Kalau kosong (tidak ada output), port itu aman dipakai. Kalau bentrok, ganti
lewat environment variable `PORT` (lihat systemd service di bawah), misal
`PORT=8282`.

### 1.3 Jalankan sebagai systemd service

Buat file `/etc/systemd/system/truf-server.service`:

```ini
[Unit]
Description=Truf 108 Online - Game Server
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/apps/truf-server
Environment=PORT=8181
ExecStart=/usr/bin/node server.js
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Aktifkan:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now truf-server
sudo systemctl status truf-server
```

Cek log kalau ada masalah: `journalctl -u truf-server -f`

### 1.4 Reverse proxy Nginx + SSL (wajib untuk WebSocket dari Vercel/HTTPS)

Karena frontend di Vercel jalan di HTTPS, backend **wajib** bisa diakses lewat
`wss://` (WebSocket Secure). Pakai Nginx + Certbot seperti pola yang biasa
Anda pakai untuk domain lain di VPS ini.

Buat `/etc/nginx/sites-available/truf-api`:

```nginx
server {
    listen 80;
    server_name truf-api.domainanda.com;

    location / {
        proxy_pass http://127.0.0.1:8181;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 86400;
    }
}
```

Aktifkan & pasang SSL:

```bash
sudo ln -s /etc/nginx/sites-available/truf-api /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d truf-api.domainanda.com
```

Ganti `truf-api.domainanda.com` dengan subdomain Anda sendiri (harus sudah
diarahkan DNS A record ke IP VPS ini).

### 1.5 Tes backend

```bash
curl https://truf-api.domainanda.com/health
# {"ok":true,"rooms":0,"uptime":...}
```

---

## 2. Frontend — deploy ke Vercel

### 2.1 Set alamat backend

Buka `frontend/index.html`, cari baris:

```js
const SERVER_URL = window.TRUF_SERVER_URL || "http://localhost:8181";
```

Cara paling praktis: **jangan ubah file ini**, cukup tambahkan baris kecil
sebelum tag `<script src="https://cdn.socket.io/...">` di `<head>` atau
sebelum `</body>`:

```html
<script>window.TRUF_SERVER_URL = "https://truf-api.domainanda.com";</script>
```

Atau langsung edit nilai default di baris tersebut jika lebih suka simpel.

### 2.2 Deploy

Cara termudah — lewat Vercel CLI:

```bash
cd frontend
npx vercel --prod
```

Ikuti prompt (login, pilih/buat project). Karena ini murni HTML statis,
Vercel akan otomatis mendeteksinya sebagai static site — tidak perlu build
step apa pun.

Atau lewat dashboard Vercel: push folder `frontend/` ke repo GitHub, lalu
"Import Project" di vercel.com, pilih repo tersebut, Framework Preset =
**Other**, Build Command kosongkan, Output Directory = `.` (root).

### 2.3 Selesai

Bagikan URL Vercel Anda (mis. `https://truf-anda.vercel.app`) ke teman-teman.
Satu orang klik **"Buat Meja"**, dapat kode 5 karakter, bagikan kode itu ke
teman lain untuk klik **"Gabung Meja"**.

---

## 3. Catatan teknis

- **Kartu lawan tidak pernah bocor** — server hanya mengirim jumlah kartu
  pemain lain ke tiap client, tangan penuh cuma dikirim ke pemiliknya.
- **Reconnect otomatis** — token pemain disimpan di `localStorage` browser;
  kalau koneksi putus/refresh, browser akan otomatis mencoba gabung kembali
  ke meja yang sama.
- **State di memori (RAM)** — kalau service backend di-restart, semua meja
  yang sedang berjalan akan hilang. Untuk pemakaian santai dengan teman ini
  biasanya tidak masalah; kalau nanti mau lebih tahan lama, bisa ditambah
  persist ke Redis/file, tapi tidak wajib untuk versi ini.
- **CORS** dibuka untuk semua origin (`*`) di server supaya gampang — cukup
  aman untuk game santai, tapi kalau mau diperketat, ganti bagian
  `cors: { origin: "*" }` di `server.js` jadi domain Vercel Anda saja.
- Minimal pemain untuk mulai: **3 orang** (target 4 atau 5 sesuai pilihan
  host saat buat meja).
