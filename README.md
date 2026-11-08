# KatalogKu API — Microservice dengan PocketBase

Backend aplikasi KatalogKu (praktikum PAWM / II3140) yang disusun sebagai **microservice**
tanpa menulis kode server. Semua tool gratis & open source:

| Komponen | Tool | Lisensi |
|---|---|---|
| product-service, account-service | [PocketBase](https://pocketbase.io) v0.40.5 | MIT |
| API gateway | Nginx 1.28 | BSD-2 |
| Orkestrasi | Docker Compose | Apache-2.0 |
| Uji API | REST Client (VS Code) / Postman / Bruno | MIT / gratis |
| Akses publik (opsional) | Cloudflare Tunnel (`cloudflared`) | Apache-2.0 |

## Arsitektur

```
 Aplikasi / Postman
        │  http://IP_SERVER:8080/api/...
        ▼
 ┌─────────────┐   /api/products    ┌──────────────────┐
 │  gateway    │ ─────────────────► │ product-service  │ → SQLite (volume product_data)
 │  (Nginx)    │   /api/auth/...    ├──────────────────┤
 │  :8080      │ ─────────────────► │ account-service  │ → SQLite (volume account_data)
 └─────────────┘   /api/favorites   └──────────────────┘
```

Setiap service punya database sendiri (database-per-service). Favorit hanya menyimpan
`product_id`; detail produk diambil aplikasi dari product-service.

## Struktur folder

```
katalogku-pocketbase/
├── docker-compose.yml        # 3 service + tunnel (opsional)
├── .env.example              # TUNNEL_TOKEN (opsional)
├── pocketbase/Dockerfile     # image PocketBase (amd64 & arm64/STB)
├── gateway/nginx.conf        # pemetaan URL /api/... → service
├── pb_schema/                # skema koleksi siap import
│   ├── product_collections.json
│   └── account_collections.json
├── seed.http                 # isi 5 produk contoh
├── api.http                  # uji semua endpoint (REST Client)
└── postman/                  # koleksi + environment Postman
```

## Menjalankan

```bash
# 1. Jalankan semua service (pertama kali butuh internet untuk build image)
docker compose up -d --build
docker compose ps

# 2. Buat superuser (admin dashboard) di kedua service
docker compose exec product-service \
  pocketbase superuser upsert admin@katalogku.id GantiPassword123 --dir=/pb_data
docker compose exec account-service \
  pocketbase superuser upsert admin@katalogku.id GantiPassword123 --dir=/pb_data
docker compose exec review-service \
  pocketbase superuser upsert admin@katalogku.id GantiPassword123 --dir=/pb_data
```

3. **Import skema** — buka dashboard, lalu *Settings → Import collections → Load from JSON file*:
   - `http://IP_SERVER:8091/_/` → `pb_schema/product_collections.json`
   - `http://IP_SERVER:8092/_/` → `pb_schema/account_collections.json`
   - `http://IP_SERVER:8093/_/` → `pb_schema/review_collections.json`
4. **Isi data contoh** — buka `seed.http` di VS Code (ekstensi REST Client), ganti
   `@pbUrl` dan password, lalu klik *Send Request* satu per satu.
5. **Uji API lewat gateway** — buka `api.http`, ganti `@baseUrl`, jalankan berurutan
   (register → login → favorit). Alternatif: import folder `postman/` ke Postman dan pilih
   environment *Lokal*, *ServerLAN*, atau *Cloudflare*.

Cek cepat: `curl http://localhost:8080/api/health` dan `curl http://localhost:8080/api/products`.

## Endpoint (lewat gateway :8080)

| Method | Endpoint | Auth | Keterangan |
|---|---|---|---|
| GET | `/api/products?perPage=30` | - | daftar produk `{ items, page, totalItems }` |
| GET | `/api/products?filter=title~'kopi'` | - | cari produk |
| GET | `/api/products/:id` | - | detail produk |
| POST | `/api/auth/records` | - | register `{ name, email, password, passwordConfirm }` |
| POST | `/api/auth/auth-with-password` | - | login `{ identity, password }` → `{ token, record }` |
| GET | `/api/favorites` | Bearer | favorit milik user login |
| POST | `/api/favorites` | Bearer | `{ user, product_id }` |
| DELETE | `/api/favorites/:recordId` | Bearer | hapus favorit |
| GET | `/api/reviews?filter=product_id='ID'` | - | review satu produk |
| POST | `/api/reviews` | Bearer | `{ product_id, user_id, user_name, rating 1-5, comment }` |

Tambah/ubah produk dilakukan admin lewat dashboard atau `seed.http` (API Rule create = superuser saja).

## Akses dari internet (opsional)

- **Quick tunnel** (tanpa akun, URL berubah tiap dijalankan):
  `docker run --rm --network host cloudflare/cloudflared:latest tunnel --no-autoupdate --url http://localhost:8080`
- **Tunnel permanen**: salin token dari dashboard Cloudflare ke `.env` (`TUNNEL_TOKEN=...`),
  lalu `docker compose --profile tunnel up -d`.

## Operasional

```bash
git pull && docker compose up -d --build            # update
docker compose logs -f account-service              # log
docker compose cp product-service:/pb_data ./backup-product   # backup
docker compose cp account-service:/pb_data ./backup-account
```

> **Keamanan:** ganti semua password contoh, jangan commit `.env`, dan jangan membuka port
> 8091/8092 ke internet — cukup gateway (8080) atau tunnel.

## Pengembangan: review-service

PocketBase #3 menyimpan rating & komentar (koleksi `reviews`, database sendiri di volume `review_data`).
Gateway memeriksa token login ke account-service (`auth_request` ke `auth-refresh`) sebelum
meneruskan POST ke review-service, sehingga hanya user yang login bisa menulis review.
Satu user hanya bisa memberi satu review per produk (unique index `user_id + product_id`).
Ubah/hapus review hanya oleh superuser.
