# nvidia-api



# 
```



```
# 
```



```
# 
```



```
# 
```


Implement fitur COMBO pada project nvidia-api.

Tujuan:
Admin dapat membuat aturan kombinasi untuk mengontrol user/API key mana yang boleh menggunakan model tertentu melalui provider tertentu dan API key provider tertentu.

Konsep:
COMBO = Client/User → Provider → Model → Provider API Key

Admin harus bisa memilih:
1. Client API Key / user
2. Provider
3. Model dari provider tersebut
4. Provider API Key yang digunakan

Aturan utama:
- Model HARUS berasal dari provider yang dipilih.
- API Key HARUS milik provider yang dipilih.
- Tidak boleh memilih model dari provider lain.
- Tidak boleh memilih API key dari provider lain.
- Routing tetap provider-locked.
- Jangan pernah fallback ke provider lain.
- Jika provider/model yang dipilih gagal, jangan pindah ke provider lain.
- Jika provider memiliki beberapa API key, yang boleh berpindah hanya antar API key milik provider yang SAMA sesuai mekanisme multi-key yang sudah ada.

Tujuan fitur ini adalah agar admin dapat mengontrol secara granular user mana menggunakan model apa, provider mana, dan credential/provider API key mana.

AUDIT SEBELUM CODING

Sebelum mengubah code, audit implementasi yang sudah ada:
- Client API Keys
- provider registry/model registry
- provider management
- provider API key storage
- provider-locked routing
- multi-key rotation
- usage attribution
- /v1/models
- authentication
- admin dashboard

Jangan membuat sistem kedua jika functionality yang diperlukan sudah tersedia. Gunakan dan perluas architecture yang sudah ada.

DESAIN DATA COMBO

Buat struktur data yang jelas, misalnya:

ComboRecord {
  id: string
  clientKeyId: string
  providerId: string
  model: string
  providerKeyId: string | null
  status: "active" | "disabled"
  createdAt: string
  updatedAt: string
  requestCount: number
  lastUsedAt: string | null
}

Jika architecture project memiliki nama/struktur yang lebih tepat, gunakan struktur existing daripada memaksakan nama di atas.

providerKeyId boleh null jika desain existing memang memiliki mode:
"gunakan semua API key provider tersebut dengan multi-key rotation".

Namun jika admin memilih API key tertentu, combo harus menggunakan credential tersebut sebagai prioritas dan tidak boleh memakai credential dari provider lain.

ADMIN UI

Tambahkan menu/tab:
"Combos"

Tampilkan daftar combo dengan informasi:
- Client/API Key
- Provider
- Model
- Provider API Key (masked)
- Status
- Request Count
- Last Used
- Created At
- Action

Action:
- Create Combo
- Enable
- Disable
- Edit
- Delete

FORM CREATE COMBO

Urutan form:

1. Pilih Client API Key/User
2. Pilih Provider
3. Setelah provider dipilih, tampilkan HANYA model yang dimiliki provider tersebut
4. Setelah provider dipilih, tampilkan HANYA provider API key milik provider tersebut
5. Simpan combo

Dynamic filtering wajib dilakukan dari registry/data backend, bukan hardcode daftar provider/model di frontend.

Contoh:
Provider = Empero
→ model dropdown hanya model Empero
→ API key dropdown hanya API key Empero

Jika Provider diganti:
→ model lama harus di-reset
→ API key lama harus di-reset
→ daftar model dan API key harus diambil ulang untuk provider baru.

VALIDASI BACKEND

Frontend filtering tidak cukup.

Backend wajib melakukan validasi ulang saat Create/Edit Combo:

- clientKeyId valid
- providerId valid
- model valid
- model memang terdaftar pada providerId
- providerKeyId jika ada memang milik providerId
- semua entity aktif
- tidak ada cross-provider reference

Jika tidak valid:
→ HTTP 400
→ jangan simpan data.

ROUTING

Saat request masuk menggunakan Client API Key:

1. Identifikasi client API key.
2. Cari combo aktif untuk client tersebut.
3. Jika combo ditemukan untuk model yang diminta:
   Client → Combo → Provider → Model → Provider API Key.
4. Request hanya dikirim ke provider tersebut.
5. Jika provider memiliki multiple API key:
   gunakan mekanisme multi-key yang sudah ada, tetapi tetap dalam provider yang sama.
6. Jika salah satu API key provider gagal/rate limited:
   boleh berpindah ke API key lain MILIK PROVIDER YANG SAMA jika mekanisme multi-key mengizinkannya.
7. JANGAN fallback ke provider lain.
8. Jika tidak ada API key provider yang dapat digunakan:
   return error yang sesuai.

Contoh:

Combo:
User A → Empero → glm-4.5 → Empero Key 2

Request:
model = glm-4.5

Routing:
User A
→ Empero
→ glm-4.5
→ Empero Key 2

Jika Key 2 gagal:
→ Key 1 Empero boleh dicoba jika multi-key aktif.

Tidak boleh:
→ NVIDIA
→ TokenHarbor
→ provider lainnya.

MODEL ACCESS CONTROL

COMBO harus menjadi salah satu sumber authorization.

Jika user/API key tidak memiliki akses ke model:
→ jangan diam-diam mencari provider lain.
→ return error model tidak diizinkan.

Jika client memiliki beberapa combo untuk model berbeda, masing-masing harus independen.

Contoh:
User A:
- glm-4.5 → Empero
- model-X → NVIDIA
- model-Y → TokenHarbor

Request glm-4.5:
→ hanya Empero.

Request model-X:
→ hanya NVIDIA.

Request model-Y:
→ hanya TokenHarbor.

Jika request model-Z yang tidak memiliki combo:
→ ditolak sesuai policy access control.
→ jangan fallback.

/v1/models

Audit endpoint /v1/models.

Untuk Client API Key:
- tampilkan hanya model yang memang diizinkan oleh access-control/combo client tersebut.
- jangan bocorkan provider internal jika API contract memang dirancang menyembunyikannya.
- jangan expose provider API key.
- jangan expose backend URL.
- jangan expose internal provider metadata yang dapat digunakan user untuk mengetahui sumber backend.

ADMIN API

Tambahkan endpoint yang diperlukan, misalnya:
GET /admin/combos
POST /admin/combos
PATCH /admin/combos/:id
DELETE /admin/combos/:id

Nama endpoint boleh mengikuti convention existing project.

Tambahkan endpoint catalog jika diperlukan:
GET /admin/combos/catalog

Catalog harus menyediakan:
- provider
- models provider
- provider API keys yang tersedia

Jangan expose secret API key asli. Hanya ID dan masked representation.

SECURITY

Sangat penting:
- Provider API key asli tidak boleh dikirim ke frontend.
- Frontend hanya menerima ID + masked value.
- Client API key user tetap hanya ditampilkan dalam bentuk masked.
- Jangan menyimpan plaintext secret baru jika architecture existing sudah menggunakan encryption/hash.
- Jangan memasukkan provider credential ke response API publik.
- Jangan membocorkan endpoint/backend URL provider kepada client.
- Jangan membuat error message yang membeberkan credential/provider secret.

USAGE

Pastikan usage tracking tetap benar.

Setiap request melalui combo harus tetap mencatat:
- client API key
- combo ID jika memungkinkan
- provider
- model
- provider key identifier
- request count
- prompt tokens
- completion tokens
- total tokens
- estimated cost
- latency
- status

Provider API key secret tidak boleh disimpan di usage log.

DASHBOARD

Jika dashboard existing memiliki usage per provider/model/API key, pastikan combo tidak merusak attribution.

Admin harus dapat mengetahui:
- combo mana yang digunakan
- user/client mana yang menggunakan
- provider/model yang digunakan
- provider key mana yang digunakan secara masked/ID
- request count
- token usage
- cost

PUBLIC API

User hanya mengetahui:
- base URL milik API kita
- API key client miliknya
- model yang diizinkan

Jangan expose:
- provider API key
- provider secret
- internal upstream URL
- credential identifier yang sensitif
- routing implementation detail.

BACKWARD COMPATIBILITY

Jangan merusak fitur yang sudah ada:
- Provider Management
- Create API Key
- provider-locked routing
- multi-key rotation
- provider cooldown
- usage tracking
- pricing
- /v1/models
- authentication
- existing providers
- streaming
- admin dashboard

Jika sistem lama memiliki Client API Key dengan allowedModels, integrasikan COMBO secara konsisten dan jangan membuat dua sumber authorization yang saling bertentangan.

Jika perlu migration, buat migration/backward-compatible handling untuk data existing.

TEST WAJIB

Tambahkan test untuk:

1. Create combo valid:
Client A + Provider A + Model A + Key A
→ SUCCESS.

2. Model provider lain:
Client A + Provider A + Model Provider B
→ 400.

3. API key provider lain:
Client A + Provider A + Model A + Key Provider B
→ 400.

4. Request dengan combo aktif:
→ routing hanya ke provider combo.

5. Provider gagal:
→ TIDAK fallback ke provider lain.

6. Provider API key gagal:
→ boleh rotate ke API key provider yang SAMA jika multi-key aktif.

7. Semua API key provider gagal:
→ error dikembalikan.

8. Combo disabled:
→ request ditolak atau mengikuti policy access-control existing, tetapi jangan fallback ke provider lain.

9. User tanpa combo:
→ tidak boleh menggunakan model yang tidak diizinkan.

10. /v1/models:
→ hanya menampilkan model yang allowed untuk client tersebut.

11. Provider/API key catalog:
→ hanya provider/model/key yang sesuai yang muncul.

12. Secret leakage test:
→ response publik tidak mengandung provider API key, upstream URL, atau secret.

13. Usage:
→ request melalui combo tetap menambah request count, tokens, cost, dan attribution dengan benar.

14. Multi-key:
→ perpindahan credential hanya terjadi dalam provider yang sama.

15. Streaming:
→ combo routing tetap benar dan usage streaming tetap tercatat.

TEST POLICY:
Jangan menggunakan atau menjalankan test Gorouter.app.
Jika test suite otomatis memuat Gorouter.app, skip/exclude test tersebut.
NVIDIA dan TokenHarbor.ai boleh diverifikasi sesuai kebutuhan.

SETELAH IMPLEMENTASI

Jalankan:
- lint
- typecheck
- build
- test yang relevan

Kemudian lakukan audit final terhadap:
- provider-locked routing
- multi-key
- access control
- secret leakage
- usage attribution
- /v1/models
- admin UI

Jangan hanya membuat UI. Pastikan COMBO benar-benar enforced di backend/routing.

Jangan menambahkan fitur lain di luar COMBO sekarang.

HASIL AKHIR YANG DIHARAPKAN:

Admin dapat membuat:

User/API Key
↓
Pilih Provider
↓
Pilih Model dari provider tersebut
↓
Pilih API Key provider tersebut
↓
Create Combo

Dan request user selalu mengikuti combo tersebut.

Yang boleh berpindah hanya API key dari provider yang SAMA.
Provider tidak boleh berubah/fallback.
```
# 
```
Audit dan perbaiki masalah USAGE TRACKING pada project nvidia-api.

Kondisi:
Dashboard di VPS 1 masih dapat dibuka, tetapi angka usage berhenti pada nilai yang sama. Prompt Tokens, Completion Tokens, Total Tokens, dan Est. Cost/Pricing tidak bertambah setelah request baru.

Source code sedang dikerjakan di VPS 2, sedangkan dashboard/service yang terlihat berjalan di VPS 1.

Jangan fokus hanya pada tampilan dashboard. Cari akar masalah mengapa request baru tidak menghasilkan penambahan usage.

Audit seluruh alur:

Request Client
→ API Gateway
→ Provider
→ Response
→ token usage extraction
→ usage calculation/pricing
→ usage storage/database
→ dashboard statistics
→ dashboard UI

Tugas:

1. Cari lokasi code yang mencatat usage setiap request.

2. Pastikan setiap request yang berhasil dan memiliki informasi token benar-benar membuat atau memperbarui usage record.

3. Audit extraction token dari response provider, termasuk response normal dan streaming jika project mendukung streaming.

Pastikan:
- prompt tokens bertambah;
- completion tokens bertambah;
- total tokens bertambah;
- request count bertambah;
- pricing/estimated cost dihitung dari usage terbaru.

4. Audit pricing calculation.

Pastikan pricing tidak berhenti menggunakan snapshot/data lama dan setiap usage baru dihitung dengan pricing/model yang benar.

Jangan mengubah angka pricing secara asal hanya agar dashboard terlihat bertambah.

5. Audit provider-specific usage format.

Provider yang berbeda dapat mengembalikan usage dengan format berbeda. Pastikan usage parser menangani semua provider yang sudah ada.

Jangan hanya memperbaiki NVIDIA jika Empero, TokenHarbor, atau provider lain memiliki format response berbeda.

6. Audit streaming.

Jika streaming response digunakan, pastikan usage tetap dicatat setelah stream selesai.

Pastikan usage tidak hilang hanya karena token usage muncul pada final chunk atau metadata tertentu.

7. Audit database/storage.

Pastikan usage record benar-benar:
- INSERT saat diperlukan;
- UPDATE jika menggunakan aggregation;
- COMMIT/persist;
- tidak hanya tersimpan di memory;
- tidak tertahan karena async callback yang tidak pernah selesai.

Periksa apakah ada error pada proses penyimpanan usage yang saat ini ditelan/silent.

8. Audit AsyncLocalStorage/request context jika project menggunakannya.

Pastikan context usage tidak hilang ketika request berpindah melalui:
- async callback;
- retry;
- provider request;
- streaming;
- worker;
- scheduler.

9. Audit fungsi seperti runWithClientKeyContext atau mekanisme context usage yang sudah ada.

Pastikan usage attribution tetap terhubung dengan Client API Key dan provider/model yang benar.

10. Audit apakah usage baru sebenarnya tersimpan tetapi dashboard membaca sumber data yang salah.

Bandingkan:
- database/storage usage terbaru;
- endpoint dashboard;
- response endpoint dashboard;
- angka yang ditampilkan UI.

Tentukan tepat di bagian mana angka berhenti bertambah.

11. Karena source code berada di VPS 2 tetapi service/dashboard berada di VPS 1, periksa kemungkinan VPS 1 menjalankan build/commit lama.

Bandingkan:
- commit/version VPS 2;
- commit/version yang sedang berjalan di VPS 1;
- build artifact;
- service/process yang menjalankan aplikasi.

Jika VPS 1 masih menjalankan code lama yang menyebabkan usage tidak tercatat, perbaiki deployment sesuai arsitektur project.

12. Jangan menghapus data usage lama.

Data yang sudah ada harus tetap dipertahankan.

13. Jangan merusak:
- Provider Management;
- Create API Key;
- multi-key;
- provider-locked routing;
- cooldown provider;
- model registry;
- authentication;
- existing usage history.

14. Buat test yang membuktikan usage benar-benar bertambah.

Minimal test:

Request 1:
prompt tokens = X
completion tokens = Y

Pastikan setelah request:
prompt tokens += X
completion tokens += Y
total tokens += X + Y
request count += 1
estimated cost bertambah sesuai pricing.

Kemudian Request 2 dengan usage berbeda.

Pastikan nilai dashboard/database menjadi akumulasi:
previous usage + request 1 + request 2.

15. Test provider yang sudah tersedia menggunakan mock response jika credential production tidak diperlukan.

Pastikan parser usage bekerja untuk provider yang berbeda.

16. Test streaming jika tersedia:
stream selesai → final usage diterima → usage disimpan → dashboard bertambah.

17. Pastikan jika usage extraction gagal, sistem mencatat error secara jelas dan tidak diam-diam menganggap request berhasil tanpa usage.

18. Setelah perbaikan, lakukan verifikasi end-to-end:

Client request
→ provider response
→ token extraction
→ pricing calculation
→ database/storage
→ dashboard API
→ dashboard UI.

Jangan hanya melihat UI. Tunjukkan bukti bahwa record/database memang bertambah.

19. Jalankan:
- lint;
- typecheck;
- build;
- test yang relevan.

Jangan menjalankan atau mengaktifkan test Gorouter.app. Jika test suite otomatis memuat Gorouter.app, skip/exclude test tersebut. NVIDIA dan TokenHarbor.ai boleh diverifikasi sesuai kebutuhan.

20. Jangan menambahkan fitur COMBO sekarang.

Fokus hanya memperbaiki USAGE TRACKING dan PRICING agar setiap request baru benar-benar menambah token, request count, dan cost.

Setelah selesai laporkan:
- akar masalah sebenarnya;
- bagian code tempat usage berhenti;
- file yang diubah;
- apakah masalah ada di token extraction, pricing, database, async context, dashboard API, deployment VPS 1, atau bagian lain;
- hasil test sebelum dan sesudah;
- bukti Request 1 dan Request 2 menghasilkan akumulasi usage;
- hasil build/typecheck/lint.

Jangan menyelesaikan masalah hanya dengan membuat UI melakukan refresh. Pastikan sumber data usage benar-benar bertambah.


```
# 
```

Lakukan AUDIT DAN PERBAIKAN menyeluruh pada project nvidia-api untuk memastikan client/user tidak dapat mengetahui atau menebak sumber backend provider melalui informasi yang dibocorkan oleh API gateway.

Fokus pada seluruh jalur API client, terutama /v1/*, bukan hanya satu endpoint.

Tujuan:
Client hanya mengetahui Base URL nvidia-api, Client API Key, model yang digunakan, dan response API yang sudah dinormalisasi. Informasi provider internal harus tetap berada di backend.

Audit dan perbaiki hal-hal berikut:

1. PROVIDER INFORMATION LEAK
Cari seluruh tempat yang mungkin membocorkan:
- nama provider
- providerId
- provider name
- upstream URL
- upstream hostname/domain
- upstream API endpoint
- credential/API key provider
- provider metadata
- internal routing information
- nama adapter/provider implementation
- informasi debug internal

Pastikan informasi tersebut tidak pernah dikirim ke client.

2. HTTP RESPONSE HEADERS
Audit semua response dari endpoint client.

Jangan meneruskan header upstream yang dapat mengungkap provider atau infrastruktur internal, termasuk header custom dari upstream.

Gunakan response header yang dikontrol oleh nvidia-api sendiri.

Pastikan header internal seperti provider/upstream/debug/tracing internal tidak ikut keluar ke client.

3. RESPONSE BODY
Audit response JSON dari seluruh endpoint client.

Jangan mengembalikan:
provider
providerId
providerName
upstream
upstreamUrl
backend
adapter
credential
internal metadata
atau informasi routing internal lainnya.

Jika provider upstream mengembalikan metadata tambahan, normalisasi/filter sebelum response dikirim ke client.

4. ERROR RESPONSE
Ini sangat penting.

Jangan meneruskan error mentah dari provider upstream karena error tersebut dapat berisi:
- nama provider
- hostname
- URL
- path internal
- nama service
- credential information
- struktur backend
- stack trace
- pesan internal

Buat error response yang aman dan tetap kompatibel dengan API client.

Contoh konsep:

Upstream:
"Empero API request failed at https://...."

Client:
error response generik yang sesuai dengan OpenAI-compatible API tanpa menyebut Empero atau URL upstream.

Tetap pertahankan HTTP status code dan informasi error yang memang aman untuk client.

5. STREAMING / SSE
Audit streaming response secara khusus.

Pastikan provider tidak dapat bocor melalui:
- SSE event
- headers
- metadata
- error event
- stream termination message
- debug information

Streaming harus tetap kompatibel dengan client tetapi tidak membocorkan sumber upstream.

6. /v1/models
Audit endpoint /v1/models.

Client hanya boleh melihat model yang memang diizinkan oleh Client API Key.

Jangan expose providerId/providerName/provider metadata.

Model harus tetap bisa digunakan normal tanpa client mengetahui provider internal.

7. CLIENT API KEY
Pastikan Client API Key tidak pernah memberikan akses untuk mengetahui provider mapping.

Jika client memiliki:
API Key → Provider → Allowed Models

maka relasi Provider tersebut harus menjadi informasi internal backend.

Client hanya mendapatkan:
API Key → Allowed Models

bukan:
API Key → Provider → Allowed Models.

8. PROVIDER ROUTING
Pertahankan aturan yang sudah diterapkan:

model → provider tetap → multi-key provider yang sama

Contoh:
GLM → Empero → Key 1 → Key 2 → Key 3

Jika semua key Empero gagal:
request gagal.

JANGAN fallback ke provider lain hanya untuk menyembunyikan error atau karena provider sedang cooldown.

9. LOGGING DAN DEBUG
Audit endpoint client dan production error handling agar stack trace, debug object, provider object, upstream request/response, dan credential tidak pernah dikirim ke client.

Internal logging boleh menyimpan informasi provider sesuai kebutuhan operasional, tetapi jangan expose internal log melalui API client.

10. CORS / BROWSER EXPOSURE
Audit response header dan mekanisme browser exposure.

Pastikan informasi internal tidak bisa diperoleh melalui exposed response headers atau endpoint publik lainnya.

11. SEARCH SELURUH CODEBASE
Cari secara menyeluruh semua penggunaan:
providerId
provider
providerName
upstream
upstreamUrl
baseUrl
endpoint
adapter
error
headers
response
metadata
debug
stack
trace

Periksa apakah ada jalur yang dapat menyebabkan data internal tersebut keluar ke client.

Jangan hanya memperbaiki file yang pertama ditemukan. Telusuri seluruh request pipeline.

12. BUAT LAYER GLOBAL
Jika memungkinkan secara arsitektur, buat satu lapisan global seperti response sanitization/provider leak protection sehingga endpoint baru yang ditambahkan di masa depan tidak mudah membocorkan informasi provider.

Namun jangan membuat duplikasi logic yang tidak diperlukan.

13. FINGERPRINTING
Tidak mungkin menjamin provider 100% tidak dapat ditebak hanya dari perilaku jaringan/model.

Tetapi minimalkan fingerprint yang berasal dari gateway sendiri:
- error format harus konsisten
- response metadata harus konsisten
- headers harus dikontrol gateway
- provider-specific metadata harus dihapus
- upstream URL/hostname tidak boleh keluar

Jangan mengubah response model secara sembarangan sampai merusak kompatibilitas API.

14. TEST WAJIB
Tambahkan atau perbaiki test untuk membuktikan:

- response normal tidak mengandung provider name
- response tidak mengandung providerId
- response tidak mengandung upstream URL
- response headers tidak membocorkan upstream
- error response tidak membocorkan provider
- error response tidak membocorkan URL/hostname
- streaming response tidak membocorkan provider
- streaming error tidak membocorkan provider
- /v1/models tidak membocorkan provider
- Client API Key tidak dapat mengetahui provider mapping
- provider internal tetap berfungsi normal
- multi-key tetap berjalan dalam provider yang sama
- provider-locked routing tetap berjalan
- fallback antar-provider tetap tidak terjadi
- existing API compatibility tetap PASS

Gunakan mock provider untuk test. Jangan menggunakan credential production.

15. KEAMANAN
Jangan menghapus informasi internal dari backend/admin Provider Management karena admin memang membutuhkan informasi tersebut.

Yang harus disembunyikan adalah informasi provider dari CLIENT API/public API.

Pisahkan dengan jelas:
ADMIN/INTERNAL → boleh melihat provider
CLIENT/PUBLIC API → tidak boleh melihat provider

16. SETELAH PERBAIKAN
Jalankan lint, typecheck, build, dan test yang relevan.

Jika menemukan kebocoran, perbaiki terlebih dahulu lalu jalankan test ulang.

Jangan hanya membuat test yang PASS tanpa memperbaiki sumber masalah sebenarnya.

Jangan menambahkan fitur COMBO sekarang.

Setelah selesai laporkan:
- file yang diaudit
- kebocoran provider yang ditemukan
- perbaikan yang dilakukan
- layer/global protection yang dibuat
- endpoint yang sudah diamankan
- test yang ditambahkan/dijalankan
- hasil lint
- hasil typecheck
- hasil build
- hasil test
- contoh response client sebelum dan sesudah jika memang ada perubahan

Jangan menjalankan atau mengaktifkan test Gorouter.app. Jika test suite otomatis memuat Gorouter.app, skip/exclude test tersebut. NVIDIA dan TokenHarbor.ai boleh diverifikasi sesuai kebutuhan.

```
# 
```
Lanjutkan project dari hasil audit sebelumnya.

Sekarang IMPLEMENTASIKAN script utama:
/root/ssh-setup

Tujuan:
Membuat satu script yang dapat dijalankan dengan satu perintah untuk mengaktifkan SSH login root menggunakan PASSWORD, termasuk jika konfigurasi cloud-init/sshd_config.d menimpa konfigurasi utama.

ATURAN PENTING:
- Jangan membuat project baru.
- Jangan menghapus konfigurasi SSH bawaan.
- Jangan mengubah file /etc/ssh/sshd_config secara langsung jika bisa dihindari.
- Gunakan drop-in configuration baru agar mudah di-rollback.
- Jangan menghapus file konfigurasi 50-cloud-init.conf atau file sshd_config.d lainnya.
- Selalu buat backup sebelum perubahan.
- Script harus idempotent: aman dijalankan berkali-kali.
- Jangan melakukan reboot server.
- Jangan mengubah firewall.
- Jangan mengubah port SSH.
- Jangan mengubah user selain root.
- Jangan memasang paket yang tidak diperlukan.
- Jangan menggunakan hardcoded password.
- Password root harus diminta secara interaktif melalui `passwd root`.
- Jangan pernah mencetak password ke terminal/log.

DESAIN SCRIPT:

1. Pastikan script dijalankan sebagai root.
   Jika bukan root:
   - tampilkan pesan error yang jelas
   - exit 1

2. Tentukan backup directory:
   /root/ssh-setup-backup

3. Buat backup timestamp sebelum perubahan, misalnya:
   /root/ssh-setup-backup/YYYYMMDD-HHMMSS/

   Backup minimal:
   - /etc/ssh/sshd_config
   - seluruh /etc/ssh/sshd_config.d/ jika tersedia

   Jangan gagal hanya karena salah satu file optional tidak ada.

4. Buat drop-in khusus untuk konfigurasi password root:
   /etc/ssh/sshd_config.d/99-root-password.conf

   Isi harus:

   PermitRootLogin yes
   PasswordAuthentication yes
   KbdInteractiveAuthentication no

   Pastikan tidak ada konfigurasi aneh atau duplikat di dalam file tersebut.

5. Jangan menghapus konfigurasi lain.

6. Pastikan root mempunyai password.
   Gunakan:
   passwd root

   Jangan pernah menerima password sebagai command-line argument.
   Jangan menyimpan password ke file.
   Jangan menampilkan password.

7. Setelah perubahan, VALIDASI konfigurasi sebelum reload/restart SSH:

   sshd -t

   Jika `sshd -t` gagal:
   - jangan reload SSH
   - tampilkan error
   - beritahu lokasi backup
   - exit 1

8. Jika validasi berhasil, tentukan apakah SSH menggunakan systemd socket activation atau service biasa.

   Periksa:
   systemctl is-active ssh.socket
   systemctl is-enabled ssh.socket

   dan:
   systemctl is-active ssh
   systemctl is-enabled ssh

   Jika ssh.socket aktif:
   - gunakan mekanisme reload/restart yang tepat
   - jangan sembarangan mematikan socket
   - pastikan port 22 tetap listen

   Jika service biasa:
   - reload SSH jika memungkinkan
   - gunakan restart hanya jika memang diperlukan

9. Setelah reload/restart, lakukan post-check:

   sshd -T

   Ambil nilai efektif:
   - permitrootlogin
   - passwordauthentication
   - kbdinteractiveauthentication
   - port

   Output harus mudah dibaca.

10. Lakukan pengecekan port SSH:

   ss -lntp | grep ':22'

   Jangan menganggap SSH berhasil hanya karena systemctl menunjukkan active.
   Pastikan port 22 benar-benar LISTEN.

11. Tampilkan hasil akhir dalam format yang jelas:

   ========================================
   SSH PASSWORD LOGIN SETUP
   ========================================

   [OK] Backup dibuat
   [OK] Drop-in configuration dibuat
   [OK] sshd configuration valid
   [OK] SSH service/socket aktif
   [OK] Port 22 LISTENING

   Effective SSH configuration:
   PermitRootLogin: ...
   PasswordAuthentication: ...
   KbdInteractiveAuthentication: ...
   Port: ...

   Backup:
   /root/ssh-setup-backup/...

   ========================================

12. Tambahkan pemeriksaan keamanan:
   - Jika PermitRootLogin efektif bukan `yes`, beri status WARNING/ERROR.
   - Jika PasswordAuthentication efektif bukan `yes`, beri status WARNING/ERROR.
   - Jika port 22 tidak LISTEN, beri status ERROR.
   - Jika semua berhasil, tampilkan `SSH PASSWORD LOGIN READY`.

13. Tambahkan mode bantuan:

   /root/ssh-setup --help

   menjelaskan:
   - fungsi script
   - lokasi backup
   - cara menjalankan
   - cara rollback

14. Tambahkan mode status:

   /root/ssh-setup --status

   Mode ini TIDAK mengubah konfigurasi.
   Hanya menampilkan konfigurasi SSH efektif dan status service/socket serta port 22.

15. Tambahkan mode rollback:

   /root/ssh-setup --rollback

   Jangan langsung menghapus sesuatu tanpa konfirmasi.

   Tampilkan backup yang tersedia, minta konfirmasi:
   "Rollback SSH configuration? [y/N]"

   Jika user memilih y:
   - pulihkan backup yang dipilih
   - jalankan `sshd -t`
   - jika valid, reload/restart SSH dengan aman
   - lakukan post-check

   Jangan rollback jika validasi gagal.

16. Buat script executable:
   chmod 700 /root/ssh-setup

17. Tambahkan shell safety:
   - gunakan bash
   - `set -Eeuo pipefail`
   - gunakan trap untuk menangani error
   - jangan membocorkan password
   - gunakan command -v untuk mengecek command penting
   - tangani sistem Ubuntu/Debian dengan baik

18. Sebelum selesai, lakukan TEST LOKAL tanpa memutus koneksi SSH saat ini.

   Test:
   - syntax shell
   - sshd -t
   - sshd -T
   - status systemd
   - port 22
   - permission script

   JANGAN menjalankan test koneksi SSH ke server dari dalam script karena bisa membuat masalah/loop.

19. Sangat penting:
   Jangan membuat script yang otomatis logout, reboot, atau memutus terminal aktif.

20. Setelah implementasi selesai, tampilkan:
   - isi lengkap `/root/ssh-setup`
   - permission file
   - hasil `--status`
   - hasil validasi sshd
   - hasil pengecekan port
   - ringkasan file yang dibuat/diubah

Jangan melakukan git commit/push.
Jangan install OpenCode/GitHub CLI.
Jangan mengubah project lain.

Kerjakan langsung implementasinya sekarang.


```
# 
```

Kita akan membuat repository terpisah bernama `ssh-setup`, khusus untuk mengaktifkan SSH password login pada VPS baru dengan cara sesederhana mungkin, idealnya satu kali menjalankan script.

Ini adalah PROMPT 1 — AUDIT & DESAIN SAJA.

Tugas:
1. Audit environment VPS saat ini:
   - OS dan versinya
   - systemd/init system
   - lokasi konfigurasi sshd
   - versi OpenSSH server
   - nama service SSH (`ssh`/`sshd`)
   - apakah root login saat ini diizinkan
   - apakah password authentication saat ini diizinkan
   - apakah ada konfigurasi di `/etc/ssh/sshd_config.d/`
   - apakah cloud-init atau konfigurasi provider berpotensi menimpa setting SSH

2. Cari semua konfigurasi SSH yang efektif dan konflik, terutama:
   - `PermitRootLogin`
   - `PasswordAuthentication`
   - `KbdInteractiveAuthentication`
   - `PubkeyAuthentication`
   - `AuthenticationMethods`

3. Jangan mengubah konfigurasi sistem.
   Jangan restart/reload SSH.
   Jangan mengubah password.
   Jangan membuat file repository.
   Jangan install package apa pun.

4. Berdasarkan hasil audit, desain script `enable-password.sh` yang nantinya:
   - aman dijalankan pada VPS baru
   - idempotent
   - membuat konfigurasi SSH khusus tanpa merusak konfigurasi bawaan
   - memvalidasi `sshd` sebelum restart
   - mendukung Ubuntu/Debian sebisa mungkin
   - menghindari lockout SSH
   - dapat dijalankan ulang tanpa menghasilkan konfigurasi duplikat
   - memberikan output/status yang jelas
   - memungkinkan root login menggunakan password

5. Periksa juga apakah ada perbedaan antara konfigurasi yang tertulis di file dan konfigurasi efektif hasil `sshd -T`.

6. Berikan laporan:
   - kondisi VPS saat ini
   - konfigurasi SSH efektif
   - potensi masalah/konflik
   - desain struktur repository
   - desain alur `enable-password.sh`
   - rekomendasi keamanan

PENTING:
- Ini hanya audit.
- Jangan melakukan perubahan apa pun pada sistem.
- Jangan commit atau push.
- Berhenti setelah laporan audit selesai.

```
# 
```

Lakukan audit dan perbaikan pada project nvidia-api dengan fokus pada cooldown/restart provider dan auto-discovery provider/model ke UI.

1. Ubah cooldown/restart provider menjadi sekitar 3 menit (180 detik). Jangan melakukan restart, retry, reset, atau recovery berulang setiap beberapa detik. Jika provider atau key mengalami error yang memang perlu cooldown, tandai cooldown sekitar 180 detik sebelum dicoba kembali.

2. Cooldown harus berlaku secara independen pada provider/key yang bermasalah. Jangan sampai provider/key yang cooldown menyebabkan request berpindah ke provider lain. Pertahankan aturan provider-locked routing:
model → provider tetap → multi-key dalam provider yang sama.

Contoh:
GLM → Empero → Key 1 → Key 2 → Key 3

Jika semua key Empero gagal atau cooldown, request harus gagal. Jangan fallback ke provider lain.

3. Audit seluruh arsitektur provider di backend dan frontend, termasuk provider registry, provider loader, configuration, model registry, initialization, routing, retry, fallback, API-key rotation, API endpoint, cache, refresh/sync, dan UI provider/model list.

4. Pastikan penambahan provider baru di code/registry otomatis terbaca oleh backend dan UI. Jangan ada daftar provider hardcoded terpisah di frontend yang harus diedit setiap kali provider baru ditambahkan.

Gunakan single source of truth untuk provider registry. Jika provider baru ditambahkan ke registry dan service melakukan load/refresh/restart sesuai arsitektur project, provider tersebut harus otomatis tersedia di backend dan otomatis muncul di UI.

5. Terapkan hal yang sama untuk model. Jika provider baru memiliki model yang terdaftar di backend/registry, model tersebut harus otomatis dapat terbaca dan ditampilkan UI tanpa harus menambahkan nama model secara manual di frontend.

6. Cari semua hardcoded provider list dan model list yang dapat menyebabkan provider sudah tersedia di backend tetapi tidak muncul di UI. Perbaiki agar frontend mengambil data provider/model dari sumber yang benar.

7. Jangan membuat hot-reload palsu. Jika restart service memang diperlukan agar provider baru terbaca, pastikan setelah restart provider otomatis ter-load dan UI mengambil data terbaru tanpa perubahan manual pada frontend. Jika dynamic reload aman dan memang didukung, gunakan mekanisme tersebut.

8. Pertahankan seluruh aturan provider-locked multi-key yang sudah diterapkan sebelumnya. Jangan mengubah routing menjadi fallback antar-provider.

9. Testing wajib dilakukan setelah coding:
- test cooldown sekitar 180 detik;
- pastikan tidak ada retry/restart setiap beberapa detik;
- key yang cooldown tidak digunakan sampai cooldown selesai;
- cooldown satu provider/key tidak memengaruhi provider lain;
- provider baru yang ditambahkan ke registry otomatis terdeteksi backend;
- provider baru otomatis muncul di UI;
- model baru otomatis muncul di UI;
- tidak ada fallback antar-provider;
- multi-key tetap berpindah hanya di dalam provider yang sama;
- existing functionality tetap bekerja.

10. Untuk pengujian auto-discovery, gunakan provider/model dummy atau mock jika diperlukan. Jangan menggunakan credential production.

11. Jangan menambahkan fitur COMBO sekarang. Fokus hanya pada cooldown/restart sekitar 3 menit, audit provider architecture, auto-discovery provider/model, sinkronisasi UI, dan provider-locked multi-key routing.

Setelah selesai, tampilkan:
- file yang diubah;
- masalah yang ditemukan saat audit;
- perubahan yang dilakukan;
- hasil test;
- bukti provider/model baru otomatis terbaca oleh UI.

Jangan hanya mengubah angka cooldown. Audit seluruh alur provider dan perbaiki jika ditemukan masalah terkait routing, registry, discovery, retry, fallback, atau sinkronisasi UI.

```
# 
```
Perbaiki sistem routing/fallback pada project `nvidia-api`.

### Tujuan utama

Terapkan aturan **provider-locked routing** untuk SEMUA provider dan SEMUA model.

Jika sebuah model sudah terhubung ke provider tertentu, request model tersebut **WAJIB tetap berada di provider tersebut**.

Yang boleh berpindah hanya **API key/account di dalam provider yang sama**.

### Aturan routing

Contoh:

`GLM → Empero`

Jika Empero memiliki beberapa key:

* Empero Key 1
* Empero Key 2
* Empero Key 3

Maka ketika request `GLM` masuk:

1. Gunakan Empero Key 1.
2. Jika key gagal karena rate limit, quota habis, authentication error, temporary provider error, atau error yang memang layak dicoba dengan key lain, pindah ke Empero Key 2.
3. Jika Key 2 gagal, lanjut ke Key 3.
4. Jika salah satu key berhasil, kembalikan response.
5. Jika SEMUA key milik Empero gagal, request harus gagal.
6. **JANGAN pernah berpindah ke provider lain sebagai fallback.**

### Larangan penting

Jangan membuat fallback seperti:

`GLM → Empero ❌ → Provider B ❌ → Provider C`

Yang benar:

`GLM → Empero → Key 1 → Key 2 → Key 3 → gagal`

Provider lain tidak boleh digunakan.

### Berlaku untuk semua provider

Jangan hardcode hanya untuk Empero atau GLM.

Implementasikan aturan ini pada arsitektur routing secara umum sehingga berlaku untuk:

* NVIDIA
* Empero
* TokenHarbor
* dan semua provider lain yang sudah ada maupun yang akan ditambahkan.

Gunakan konsep:

`model → provider → keys`

Bukan:

`model → list semua provider → fallback provider`

### Multi-key

Pertahankan/implementasikan mekanisme multi-key per provider.

Setiap provider dapat mempunyai beberapa credential/key.

Routing key boleh menggunakan mekanisme yang sudah ada seperti round-robin atau mekanisme key rotation yang sesuai.

Yang penting:

**key rotation hanya terjadi di dalam provider yang sama.**

### Error handling

Bedakan antara:

* error pada satu key → boleh mencoba key lain dari provider yang sama
* semua key provider gagal → return error
* model tidak tersedia pada provider tersebut → return error
* provider tidak tersedia → return error

Jangan menangkap error lalu diam-diam mengirim request ke provider lain.

### Provider mapping

Jangan merusak mapping model/provider yang sudah ada.

Jika saat ini:

`GLM → Empero`

maka tetap:

`GLM → Empero`

Jangan mengubahnya menjadi provider fallback chain.

Jika ada model yang memang dikonfigurasi memiliki beberapa provider secara eksplisit, pertahankan konfigurasi tersebut hanya jika arsitektur project memang sudah mendukung konsep tersebut. Jangan menganggap semua provider yang mendukung nama model sebagai fallback otomatis.

### Arsitektur

Cari seluruh bagian code yang menangani:

* model selection
* provider selection
* fallback
* retry
* API key rotation
* provider registry
* request dispatch
* error handling

Pastikan tidak ada jalur tersembunyi yang melakukan fallback antar-provider.

Buat logic provider-locking di level routing/core agar aturan ini berlaku konsisten untuk seluruh provider.

Contoh pseudocode:

```text
model = requested_model

provider = resolveProvider(model)

keys = provider.getKeys()

for key in rotate(keys):
    response = request(provider, key, model)

    if success:
        return response

    if retryable_key_error:
        continue

    return error

return provider_error

Yang TIDAK boleh:

text
for provider in providers:
    try:
        request(provider)

### Testing

Setelah perubahan selesai, buat/jalankan test yang membuktikan:

1. Model hanya menggunakan provider yang sudah ditentukan.
2. Key 1 gagal → Key 2 digunakan.
3. Key 2 gagal → Key 3 digunakan.
4. Key 3 berhasil → response berhasil.
5. Semua key gagal → request gagal.
6. Semua key gagal → TIDAK ada provider lain yang dicoba.
7. Provider lain yang kebetulan mendukung model tersebut tidak digunakan sebagai fallback otomatis.
8. Aturan yang sama berlaku untuk provider lain, bukan hanya Empero.
9. Existing functionality tidak rusak.

Gunakan mock/test provider bila diperlukan agar test tidak membutuhkan credential production.

### Penting

Jangan menambahkan fitur **combo** sekarang.

Fokus perubahan kali ini hanya:

**Provider tetap → multi-key rotation di dalam provider tersebut → gagal jika semua key provider tersebut gagal.**

Setelah implementasi selesai, tampilkan:

* file yang diubah
* perubahan utama
* hasil test
* contoh flow routing sebelum dan sesudah perubahan

Jangan berhenti hanya setelah coding. Pastikan implementasi benar-benar diverifikasi dengan test.



```
# 
```
PROMPT — IMPLEMENT ADMIN UI FROM DESIGN SPEC

Project: nvidia-api

Design reference sekarang sudah diterjemahkan menjadi specification tekstual.

SOURCE OF TRUTH:
docs/design/ADMIN-DESIGN-SPEC.md

PENTING:
Jangan mencoba menebak desain dari screenshot.
Jangan membuat desain baru.
Ikuti ADMIN-DESIGN-SPEC.md secara ketat.

TUJUAN:
Implementasikan ulang UI Admin Dashboard agar mengikuti design specification yang sudah dibuat.

Fokus utama:
- Sidebar
- Topbar
- Provider Management
- Provider Cards
- API Key Management
- Model Registry
- Usage Dashboard
- Usage Logs
- Pricing Management
- Backup & Restore
- System Settings

ATURAN:

1. LAYOUT

Gunakan struktur:

Sidebar kiri
+
Topbar
+
Main Content

Sidebar:
- 248px desktop
- sticky
- responsive drawer pada layar kecil

Topbar:
- sticky
- minimal 58px
- breadcrumb
- search
- refresh
- system status
- admin profile

Main content:
- max-width 1400px
- padding 24px

2. SIDEBAR

Gunakan menu:

Overview
Providers
API Key Management
Model Registry
Usage Dashboard
Usage Logs
Pricing Management
Backup & Restore
System Settings

Pastikan:
- active state jelas
- hover state
- icon konsisten
- tidak ada menu duplicate yang tidak diperlukan
- sidebar menjadi primary navigation

3. PROVIDER MANAGEMENT

Provider Management harus mengikuti spec.

Desktop:
3 kolom

Tablet:
2 kolom

Mobile:
1 kolom

Provider card harus memiliki:

- provider icon
- provider name
- provider domain
- ENABLED/DISABLED badge
- Models
- API Keys
- Requests
- Manage API Keys
- Enable/Disable
- jumlah model tambahan

Gunakan data provider existing.

JANGAN menggunakan dummy provider.

4. DESIGN TOKEN

Gunakan design token dari:

docs/design/ADMIN-DESIGN-SPEC.md

Jangan membuat warna baru di luar design system kecuali benar-benar diperlukan.

Gunakan:
- primary blue
- success green
- danger red
- light background
- border
- muted text
- card shadow
- radius sesuai spec

UI harus terang/clean.

JANGAN menggunakan dark dashboard.

5. COMPONENT REUSE

Audit component existing terlebih dahulu.

Jika sudah ada:
- Button
- Card
- Badge
- Input
- Modal
- Table
- Sidebar
- Header
- Pagination
- Empty state
- Error state
- Skeleton

gunakan kembali.

Jangan membuat component duplicate.

6. FUNCTIONALITY

UI hanya mengubah presentation layer.

JANGAN merusak:
- Provider Management
- Enable/Disable Provider
- API Key Management
- Model Registry
- Usage Tracking
- Usage Dashboard
- Logs
- Pricing
- Backup/Restore
- System Settings
- `/v1/models`
- API request
- streaming

Semua data harus tetap berasal dari API/service/storage existing.

7. API KEY MANAGEMENT

Pertahankan security behavior existing.

API key:
- jangan ditampilkan plaintext setelah disimpan
- gunakan masked representation
- raw key hanya boleh diterima melalui flow create yang sudah ada
- jangan memasukkan secret ke frontend log
- jangan menyimpan credential di localStorage jika arsitektur existing tidak menggunakannya

8. USAGE DASHBOARD

Gunakan component chart existing jika sudah tersedia.

Jika chart component belum ada:
buat reusable chart component berdasarkan design token.

Jangan membuat chart hanya sebagai dekorasi.

Data harus berasal dari Usage API existing.

9. LOGS

Gunakan:
- table
- filter
- pagination
- status badge
- provider
- model
- HTTP status
- token usage
- latency

Pastikan credential tetap masked.

10. RESPONSIVE

WAJIB diuji pada:

Desktop
Tablet
Mobile

Mobile:
- sidebar menjadi drawer
- topbar tetap usable
- provider card menjadi satu kolom
- table dapat di-scroll atau menggunakan responsive layout
- button tidak keluar layar
- tidak ada horizontal overflow yang tidak diperlukan

11. VISUAL QUALITY

Target desain:

clean
modern
professional
light
premium
minimal

Hindari:
- gradient berlebihan
- dark background
- card terlalu gelap
- shadow berat
- border berlebihan
- typography terlalu besar
- menu duplicate

12. IMPLEMENTATION PROCESS

Sebelum coding:

1. Audit frontend existing.
2. Identifikasi routing.
3. Identifikasi component system.
4. Identifikasi styling system.
5. Identifikasi API calls.
6. Identifikasi halaman admin yang sudah ada.

Kemudian implementasikan design system secara bertahap.

Prioritas:

Phase 1:
- App shell
- Sidebar
- Topbar

Phase 2:
- Provider Management

Phase 3:
- API Key Management
- Model Registry

Phase 4:
- Usage Dashboard
- Logs

Phase 5:
- Pricing
- Backup/Restore
- System Settings

Jangan menghapus halaman yang sudah ada.

13. TESTING

Setelah implementasi:

npm run lint
npm run build
npm test

Jika tersedia:
npm run dev

Lakukan juga pemeriksaan frontend untuk:
- route tidak rusak
- API request tidak berubah
- provider toggle tetap bekerja
- API key management tetap bekerja
- usage tetap tampil
- logs tetap tampil
- backup/restore tetap bekerja

Jangan menjalankan test Gorouter.app.

14. JANGAN MELAKUKAN

Jangan:
- membuat backend baru
- membuat database baru
- membuat provider dummy
- membuat API dummy
- mengubah business logic
- mengganti endpoint existing
- menghapus fitur
- melakukan refactor besar
- mengubah credential
- membocorkan API key
- mengubah test hanya agar lulus

15. HASIL AKHIR

Laporkan:

- file frontend yang diubah
- component baru
- component yang digunakan kembali
- halaman yang sudah mengikuti design spec
- responsive behavior
- design token yang digunakan
- hasil lint
- hasil build
- hasil test
- jumlah test pass/fail/skip
- masalah yang masih tersisa

PENTING:

ADMIN-DESIGN-SPEC.md adalah SOURCE OF TRUTH.

Jika ada perbedaan antara UI lama dan specification:
ikuti specification untuk visual/UI,
tetapi pertahankan functionality dan backend existing.


```
# 
```
PROMPT — CREATE ADMIN DASHBOARD DESIGN SPECIFICATION

Project: nvidia-api

Saya memiliki referensi desain Admin Dashboard yang harus menjadi standar UI project.

File referensi:
docs/design/admin-dashboard-reference.png

IMPORTANT:
Environment/AI yang menjalankan task ini mungkin TIDAK memiliki kemampuan vision untuk membaca isi gambar.

Karena itu JANGAN menganggap AI dapat memahami desain hanya dengan membaca file PNG.

Tugas sekarang BUKAN melakukan redesign.

Tugas hanya:
menganalisis struktur frontend existing dan membuat dokumentasi design specification tekstual yang dapat digunakan AI coding tanpa perlu melihat gambar.

Buat file:

docs/design/ADMIN-DESIGN-SPEC.md

Dokumen tersebut harus menjadi sumber kebenaran (source of truth) untuk desain Admin Dashboard.

Dokumen WAJIB menjelaskan secara detail:

1. GLOBAL DESIGN
- light theme
- background colors
- primary blue
- text colors
- secondary text
- success green
- danger red
- border colors
- card colors
- shadow
- border radius
- typography hierarchy

2. PAGE LAYOUT
Jelaskan struktur:

Sidebar kiri
+
Top Header
+
Main Content

Jelaskan:
- sidebar width
- header height
- content padding
- gap
- responsive behavior

Gunakan nilai CSS yang konkret jika dapat ditentukan.

3. SIDEBAR

Dokumentasikan:
- posisi
- ukuran
- background
- border
- logo
- branding
- menu
- icon
- active state
- hover state
- spacing
- system status card
- version footer

Menu:

Overview
Providers
API Key Management
Model Registry
Usage Dashboard
Usage Logs
Pricing Management
Backup & Restore
System Settings

4. HEADER

Dokumentasikan:
- sidebar toggle
- breadcrumb
- search
- Ctrl+K indicator
- refresh
- system status
- admin profile
- spacing
- alignment

5. PROVIDER MANAGEMENT

Dokumentasikan:
- page title
- subtitle
- Add Provider button
- provider grid
- desktop 3 columns
- tablet 2 columns
- mobile 1 column

6. PROVIDER CARD

Dokumentasikan secara detail:

Header:
provider icon
provider name
provider domain
enabled badge

Statistics:
Models
API Keys
Requests

Actions:
Manage API Keys
Disable/Enable
additional model count

Jelaskan posisi setiap elemen.

7. COLORS

Gunakan color token.

Contoh format:

--color-primary
--color-primary-hover
--color-primary-light
--color-text-primary
--color-text-secondary
--color-border
--color-success
--color-success-light
--color-danger
--color-danger-light
--color-background

Gunakan HEX/RGB yang realistis berdasarkan desain reference.

8. BUTTONS

Dokumentasikan:

Primary
Secondary
Danger
Icon button

Jelaskan:
- height
- padding
- radius
- font size
- border
- hover
- active
- disabled

9. BADGES

Dokumentasikan ENABLED badge dan status lainnya.

10. CARDS

Dokumentasikan:
- background
- border
- radius
- shadow
- padding
- spacing

11. TYPOGRAPHY

Dokumentasikan:
- page title
- section title
- card title
- body
- metadata
- button
- badge

12. ICONS

Dokumentasikan:
- ukuran
- style
- alignment
- warna
- penggunaan icon library existing jika tersedia.

13. RESPONSIVE

Dokumentasikan behavior:

Desktop
Tablet
Mobile

Termasuk:
- sidebar collapse
- mobile drawer
- card grid
- header
- search
- tables
- buttons

14. OTHER ADMIN PAGES

Design specification juga harus berlaku untuk:

Overview
API Key Management
Model Registry
Usage Dashboard
Usage Logs
Pricing Management
Backup & Restore
System Settings

Semua harus menggunakan design system yang sama.

15. NAVIGATION RULE

Sidebar = primary navigation.

Header = global controls/context.

Jangan membuat menu navigasi duplicate di header.

16. FUNCTIONALITY

Design specification TIDAK boleh mengubah business logic.

UI redesign tidak boleh merusak:
- Provider Management
- API Key Management
- Model Registry
- Usage
- Logs
- Pricing
- Backup/Restore
- Settings
- API endpoints.

17. COMPONENT SYSTEM

Identifikasi component existing yang dapat digunakan kembali.

Jika sudah ada:
- Button
- Card
- Badge
- Input
- Modal
- Table
- Sidebar
- Header

gunakan kembali.

Jangan membuat component duplicate tanpa alasan.

18. IMPLEMENTATION RULE

Jangan melakukan coding redesign pada task ini.

Hanya:
- audit frontend existing
- dokumentasikan design system
- dokumentasikan layout
- dokumentasikan component
- dokumentasikan responsive behavior

Setelah selesai, tampilkan:

- lokasi file specification
- ringkasan design token
- struktur layout
- component yang ditemukan
- component yang belum tersedia
- rekomendasi implementasi tahap berikutnya

Jangan mengubah backend.
Jangan mengubah business logic.
Jangan mengubah API.
Jangan melakukan refactor besar.


```

# Prompt — Implementasi UI Mengikuti Design Reference
```
PROMPT — IMPLEMENT ADMIN DASHBOARD UI SESUAI DESIGN REFERENCE

Project: `nvidia-api`

Gunakan file design reference yang sudah disimpan sebelumnya:

`docs/design/admin-dashboard-reference.png`

Gambar tersebut adalah REFERENSI VISUAL UTAMA dan harus menjadi acuan utama untuk redesign Admin Dashboard.

TUJUAN:
Ubah UI Admin Dashboard `nvidia-api` agar mengikuti desain pada reference secara konsisten.

PENTING:
Jangan hanya mengganti warna.
Ikuti struktur layout, spacing, ukuran komponen, typography, card, button, navigation, icon, status badge, dan hierarchy visual seperti pada reference.

==================================================
1. DESIGN STYLE
==================================================

Gunakan LIGHT THEME.

Karakter utama:
- background putih / sangat terang
- primary color blue
- dark navy untuk heading/text utama
- secondary text abu-abu/biru muda
- card putih
- border tipis
- shadow sangat ringan
- border radius modern
- spacing lega
- tampilan clean
- profesional
- modern SaaS/Admin Dashboard
- tidak menggunakan dark background
- jangan menggunakan gradient berlebihan
- jangan menggunakan efek glow berlebihan

Primary blue harus menjadi warna utama action dan navigation aktif.

Status:
- ENABLED / Operational → hijau
- Disabled / Error → merah
- Warning → amber/orange
- Information → blue

==================================================
2. GLOBAL LAYOUT
==================================================

Gunakan layout seperti reference:

┌──────────────────────────────────────────────────────┐
│ SIDEBAR │ HEADER / TOP BAR                           │
│         ├────────────────────────────────────────────┤
│         │ PAGE CONTENT                               │
│         │                                            │
│         │                                            │
└──────────────────────────────────────────────────────┘

SIDEBAR:
- fixed/sticky di sisi kiri
- lebar konsisten
- background putih
- border kanan tipis
- logo dan branding di bagian atas
- menu vertical
- active menu menggunakan background biru sangat muda + indicator biru
- icon + text
- spacing antar menu rapi

HEADER:
- berada di sebelah kanan sidebar
- background putih
- border bawah tipis
- hamburger/sidebar toggle
- breadcrumb
- search
- refresh
- system status
- admin profile

CONTENT:
- berada di sebelah kanan sidebar
- padding konsisten
- responsive
- tidak terlalu rapat dengan sidebar/header

==================================================
3. SIDEBAR
==================================================

Ikuti struktur reference.

Branding:

nvidia-api
ADMIN DASHBOARD

Gunakan logo/icon sederhana yang profesional.

Menu utama:

Overview
Providers
API Key Management
Model Registry
Usage Dashboard
Usage Logs
Pricing Management
Backup & Restore
System Settings

Setiap menu:
- icon
- label
- hover state
- active state
- disabled state jika diperlukan

ACTIVE MENU:
- text blue
- icon blue
- background light blue
- indicator biru pada sisi kiri
- border radius sesuai reference

JANGAN membuat sidebar terlalu gelap.

==================================================
4. HEADER
==================================================

Header mengikuti reference.

Komponen:

A. Sidebar Toggle
- button kecil
- rounded
- icon hamburger

B. Breadcrumb

Contoh:

Providers  >  Provider Management

C. Search

Placeholder:

Search...

Shortcut:

Ctrl + K

Search harus terlihat seperti input/button modern dengan icon search.

D. Refresh Button
- icon refresh
- compact
- tooltip bila diperlukan

E. System Status

Contoh:

● System Operational

Gunakan:
- green indicator
- text dark
- border ringan

F. Admin Profile

Contoh:

A
admin
Administrator
⌄

Gunakan avatar lingkaran biru.

==================================================
5. PROVIDER MANAGEMENT PAGE
==================================================

Halaman Provider Management harus mengikuti reference.

Header:

Provider Management

Subtitle:

Manage and configure your AI providers and their API connections.

Di kanan:

+ Add Provider

Button:
- primary blue
- white text
- rounded
- icon plus
- ukuran compact seperti reference

==================================================
6. PROVIDER CARD
==================================================

Provider card adalah bagian penting.

Gunakan grid:

Desktop:
3 kolom

Tablet:
2 kolom

Mobile:
1 kolom

Card:
- background putih
- border tipis
- radius modern
- shadow sangat ringan
- padding konsisten
- tinggi card seragam
- jangan membuat card terlalu besar

Struktur card:

┌─────────────────────────────────────┐
│ ICON  Provider Name     ENABLED     │
│       provider.domain               │
│                                     │
│ ◇ Models   🔧 API Keys   ⚡ Requests │
│                                     │
│ [Manage API Keys] [Disable] + 1030 │
└─────────────────────────────────────┘

Provider header:
- provider icon/logo
- provider name
- provider domain
- status badge

Status badge:

● ENABLED

Gunakan:
- green text
- light green background
- green border

==================================================
7. PROVIDER STATISTICS
==================================================

Setiap provider card menampilkan:

Models
API Keys
Requests

Contoh:

1050
Models

9
API Keys

1800
Requests

Gunakan icon berbeda untuk setiap statistik.

Models:
- cube icon
- blue

API Keys:
- wrench/key icon
- blue

Requests:
- lightning icon
- green

Buat ketiga statistik sejajar dan mudah dibaca.

==================================================
8. PROVIDER ACTIONS
==================================================

Bagian bawah provider card:

[ Manage API Keys ]

[ Disable ]

+ 1030 models

Manage API Keys:
- outline blue
- white/light background
- blue text

Disable:
- outline merah
- red text
- jangan menggunakan solid red sebagai default

Jumlah model tambahan:
- blue text
- aligned ke kanan

Jika provider disabled:

[ Enable ]

Gunakan style yang konsisten.

JANGAN menghapus data provider/model ketika Disable.

==================================================
9. PROVIDER ICON
==================================================

Gunakan icon/logo provider yang tersedia secara aman.

Jika provider memiliki logo:
- tampilkan logo

Jika tidak memiliki logo:
- gunakan fallback icon/avatar yang konsisten.

Jangan membuat logo palsu yang menyerupai trademark secara tidak perlu.

Pastikan icon:
- ukuran konsisten
- posisi konsisten
- tidak merusak layout

==================================================
10. SYSTEM STATUS CARD
==================================================

Sidebar bagian bawah mengikuti reference.

Card:

System Status

● Operational

Uptime

7d 14h 32m

Gunakan:
- white background
- border
- rounded
- padding
- green status indicator

Data harus berasal dari backend/runtime jika sudah tersedia.

Jangan hardcode uptime.

==================================================
11. FOOTER / SIDEBAR VERSION
==================================================

Tampilkan:

© 2024 nvidia-api
v1.0.0

Tetapi jika project sudah memiliki version aktual:
gunakan version aktual dari project.

Jangan hardcode versi jika backend/package.json sudah menyediakan versi.

==================================================
12. INFO BAR
==================================================

Di bawah provider grid, tambahkan information bar seperti reference:

ⓘ Click on “Manage API Keys” to view and manage the API keys for each provider.

Gunakan:
- background sangat light blue
- border tipis
- icon information
- text dark blue
- rounded corners

==================================================
13. API KEY MANAGEMENT
==================================================

Semua halaman API Key Management juga harus mengikuti design system yang sama.

Jangan membuat halaman API Key dengan desain berbeda.

Gunakan:
- light background
- card putih
- blue primary action
- red danger action
- rounded input
- clean table
- status badge
- consistent spacing

SECURITY:

Raw API key/provider credential JANGAN ditampilkan.

Gunakan masking.

Contoh:

sk-••••••••••••••••1234

Jangan menampilkan secret penuh.

==================================================
14. MODEL REGISTRY
==================================================

Model Registry harus mengikuti design reference.

Gunakan:
- white cards/table
- blue headings
- clean filters
- provider badge
- model name
- model ID
- status
- actions

Jangan mengubah data/model registry hanya untuk kebutuhan UI.

==================================================
15. USAGE DASHBOARD
==================================================

Usage Dashboard juga harus mengikuti design system yang sama.

Gunakan cards untuk:

Total Requests
Successful
Failed
Blocked
Input Tokens
Output Tokens
Total Tokens
Average Latency

Gunakan hierarchy yang jelas.

Jangan membuat dashboard dark.

Chart jika sudah tersedia:
- gunakan style clean
- background putih
- blue sebagai primary
- green untuk success
- red untuk error
- grid/chart tidak terlalu ramai.

==================================================
16. USAGE LOGS
==================================================

Usage Logs harus menggunakan table modern seperti SaaS dashboard.

Kolom minimal:

Timestamp
Provider
Model
Status
HTTP Status
Input Tokens
Output Tokens
Total Tokens
Latency

Status menggunakan badge:

SUCCESS
ERROR
BLOCKED

Gunakan pagination.

Jangan menampilkan API key/credential secara penuh.

==================================================
17. PRICING MANAGEMENT
==================================================

Pricing Management harus menggunakan design system yang sama.

Gunakan:
- clean cards/table
- blue primary action
- input modern
- status badge
- edit/delete action

Jangan mengubah logic pricing yang sudah ada.

UI saja yang disesuaikan jika tidak diperlukan perubahan backend.

==================================================
18. BACKUP & RESTORE
==================================================

Backup & Restore mengikuti design system yang sama.

Gunakan:
- backup cards/table
- timestamp
- size
- record count
- version
- status
- Backup button
- Restore button
- Delete button jika fitur existing memang ada

Restore harus tetap memiliki confirmation dialog.

Jangan mengubah mekanisme backup/restore hanya karena redesign UI.

==================================================
19. SYSTEM SETTINGS
==================================================

System Settings mengikuti style yang sama.

Gunakan section/card:

General
Security
Provider
Usage
System

Gunakan:
- labels
- descriptions
- switches
- buttons
- clean form controls

==================================================
20. RESPONSIVE
==================================================

WAJIB responsive.

Desktop:
- sidebar tetap terlihat
- provider grid 3 kolom

Tablet:
- sidebar dapat collapse
- provider grid 2 kolom

Mobile:
- sidebar menjadi drawer
- provider grid 1 kolom
- header menyesuaikan
- card tidak overflow
- table menjadi responsive/scroll jika diperlukan

Tidak boleh ada:
- horizontal overflow yang tidak diperlukan
- card terpotong
- button keluar layar
- text overlap

==================================================
21. NAVIGATION RULE
==================================================

INI SANGAT PENTING.

Sidebar dan Header TIDAK BOLEH memiliki fungsi navigasi yang sama.

SIDEBAR:
→ navigasi halaman utama aplikasi.

HEADER:
→ global controls dan context:
- sidebar toggle
- breadcrumb
- search
- refresh
- system status
- profile

Jangan membuat duplicate menu navigation di header.

==================================================
22. DESIGN CONSISTENCY
==================================================

Semua halaman harus terlihat sebagai SATU aplikasi.

Gunakan design token/komponen reusable untuk:

- colors
- typography
- spacing
- border radius
- shadows
- buttons
- inputs
- cards
- badges
- tables
- modals
- icons
- navigation

Jangan membuat style berbeda-beda per halaman.

Jika project menggunakan Tailwind/CSS variables/component system:
gunakan sistem existing tersebut.

Jangan membuat CSS duplicate jika component existing dapat digunakan kembali.

==================================================
23. FUNCTIONALITY PRESERVATION
==================================================

PENTING:

Redesign UI TIDAK BOLEH merusak functionality.

Pertahankan:

- Provider Management
- Enable/Disable Provider
- API Key Management
- Model Registry
- `/v1/models`
- Usage Tracking
- Usage Dashboard
- Usage Logs
- Pricing
- Backup
- Restore
- Admin API
- streaming
- existing provider integrations

Jangan mengubah API/backend hanya untuk mempercantik UI jika tidak diperlukan.

==================================================
24. DATA HARUS REAL
==================================================

Jangan menggunakan:

- dummy provider
- dummy model
- fake request count
- fake API key count
- fake uptime
- fake token usage
- fake pricing

Semua data harus berasal dari backend/API/storage existing.

Jika data belum tersedia:
- tampilkan empty state yang baik
- jangan mengarang angka.

==================================================
25. ACCESSIBILITY
==================================================

Pastikan:

- button memiliki label
- icon-only button memiliki tooltip/aria-label
- contrast text cukup
- focus state terlihat
- keyboard navigation dapat digunakan
- form input memiliki label
- modal dapat ditutup dengan keyboard jika existing system mendukungnya

==================================================
26. PERFORMANCE
==================================================

Jangan membuat redesign menyebabkan:

- request API tambahan yang tidak diperlukan
- polling berlebihan
- rendering provider berulang tanpa alasan
- loading seluruh logs sekaligus

Gunakan API existing.

==================================================
27. IMPLEMENTATION RULE
==================================================

Sebelum coding:

1. Audit struktur frontend existing.
2. Temukan entry point Admin Dashboard.
3. Temukan routing.
4. Temukan component/layout existing.
5. Temukan CSS/design system existing.
6. Temukan API client yang digunakan UI.

Kemudian implementasikan redesign dengan memanfaatkan struktur existing.

Jangan membuat frontend kedua.

Jangan membuat duplicate Admin Dashboard.

Jangan mengganti framework existing.

Jangan melakukan refactor besar yang tidak diperlukan.

==================================================
28. REFERENCE MATCH
==================================================

Gunakan:

`docs/design/admin-dashboard-reference.png`

sebagai visual reference.

Target visual:

- layout sangat mirip reference
- light theme
- sidebar kiri
- header atas
- content area luas
- provider cards 3 kolom
- blue primary
- white cards
- green enabled badge
- red disable button
- clean typography
- light borders
- subtle shadows
- modern spacing

Tidak perlu menyalin pixel secara buta, tetapi hasil akhir harus jelas terlihat sebagai implementasi dari reference tersebut.

==================================================
29. VALIDATION
==================================================

Setelah implementasi:

Jalankan:

npm run lint
npm run build
npm test

Jika ada test Gorouter.app:

JANGAN menjalankan atau memicu test/integration test Gorouter.app.

Jika test suite otomatis memuat test Gorouter:
- skip/exclude test tersebut
- jangan mengubah test agar terlihat lulus
- laporkan jumlah test yang dijalankan dan jumlah yang di-skip.

Jangan menggunakan Gorouter.app sebagai fallback/provider untuk verification.

Jangan melakukan live provider test NVIDIA/TokenHarbor hanya untuk kebutuhan redesign UI.

==================================================
30. FINAL UI CHECK
==================================================

Setelah selesai, periksa secara manual/automated jika memungkinkan:

- sidebar
- header
- breadcrumb
- search
- refresh
- system status
- profile
- provider cards
- provider statistics
- enable/disable
- Add Provider
- API Key Management
- Model Registry
- Usage Dashboard
- Usage Logs
- Pricing
- Backup/Restore
- Settings
- responsive desktop
- responsive tablet
- responsive mobile

Pastikan tidak ada:
- dark theme
- duplicate navigation
- layout rusak
- overflow
- button overlap
- text overlap
- fake data
- exposed API key
- broken functionality

==================================================
31. FINAL REPORT
==================================================

Setelah selesai laporkan:

1. File yang diubah.
2. Component/layout yang dibuat atau diperbaiki.
3. Design system yang digunakan.
4. Apakah UI sudah mengikuti reference.
5. Halaman yang sudah disesuaikan.
6. Responsive status.
7. Functionality yang diverifikasi.
8. `npm run lint`
9. `npm run build`
10. `npm test`
11. Jumlah test pass/fail/skip.
12. Test Gorouter yang di-skip jika ada.
13. Masalah yang masih tersisa.

PENTING:
Jangan hanya membuat Provider Management terlihat bagus.

Seluruh Admin Dashboard harus menggunakan design system yang sama seperti reference.

Jangan mengubah backend/business logic kecuali benar-benar diperlukan untuk mempertahankan kompatibilitas UI.

Jangan membuat data palsu.

Jangan membuat duplicate navigation.

Jangan menggunakan dark theme.

Gunakan `docs/design/admin-dashboard-reference.png` sebagai sumber referensi visual utama selama proses implementasi.


```
# 
```

PROMPT — SAVE ADMIN UI DESIGN REFERENCE

Pada project `nvidia-api`, saya lampirkan gambar desain UI Admin Dashboard yang akan menjadi DESIGN REFERENCE utama.

TUGAS:

1. Simpan gambar desain tersebut ke project dengan lokasi:
   `docs/design/admin-dashboard-reference.png`

2. Jika folder belum ada, buat:
   `docs/design/`

3. Jangan mengubah atau menghapus source code existing.

4. Buat file dokumentasi:
   `docs/design/README.md`

Isi README menjelaskan bahwa:

- `admin-dashboard-reference.png` adalah referensi visual utama untuk Admin Dashboard.
- Semua pengembangan UI berikutnya harus mengikuti desain tersebut.
- Gunakan desain sebagai acuan:
  - warna
  - spacing
  - typography
  - card
  - button
  - sidebar
  - header
  - provider cards
  - status badge
  - layout
  - responsive behavior
  - border radius
  - icon style
  - hierarchy informasi

5. PENTING:
   Desain menggunakan LIGHT THEME, bukan dark theme.

   Karakter visual:
   - background putih / sangat terang
   - primary blue
   - teks dark navy
   - card putih
   - border tipis
   - shadow sangat ringan
   - status success hijau
   - status danger merah
   - tampilan modern, clean dan profesional
   - tidak menggunakan background gelap sebagai tema utama

6. Struktur navigasi pada desain harus dipahami sebagai:
   Sidebar = navigasi utama aplikasi.
   Header = kontrol halaman/global seperti search, refresh, system status dan admin profile.

   JANGAN membuat menu sidebar dan menu header menjadi dua navigasi yang memiliki fungsi sama.

7. Jangan langsung mengubah UI existing hanya karena gambar sudah disimpan.

   Tahap ini hanya:
   - menyimpan reference
   - mendokumentasikan design system
   - memastikan file dapat digunakan AI sebagai referensi pada task berikutnya.

8. Setelah selesai:
   - cek file benar-benar tersimpan
   - cek path file
   - pastikan PNG dapat dibaca
   - jangan membuat duplicate reference image
   - jalankan lint/build jika perubahan dokumentasi/source membutuhkan validasi

9. JIKA ADA MASALAH:
   Jangan hanya melaporkan masalah.

   Jika ada error yang disebabkan perubahan yang kamu lakukan:
   → langsung perbaiki
   → jalankan ulang validasi
   → laporkan hasil akhirnya.

HASIL AKHIR:
Laporkan:
- lokasi design reference
- lokasi README design
- apakah gambar berhasil dibaca
- apakah ada file source yang berubah
- hasil validasi
- masalah yang ditemukan dan langsung diperbaiki jika ada.

Jangan membuat desain baru pada tahap ini.
Gunakan gambar yang saya lampirkan sebagai reference resmi.

```
# Prompt: API Key Management Final UI & E2E
```

Lanjutkan project `nvidia-api`.

JANGAN hanya audit.
Jika menemukan bug, langsung identifikasi root cause, PERBAIKI source code, tambahkan regression test, lalu jalankan ulang validation.

FOKUS:
Finalisasi fitur API Key Management melalui UI sampai benar-benar end-to-end.

FITUR YANG WAJIB DIVERIFIKASI DAN DIPERBAIKI JIKA SALAH:

1. API KEY LIST

Pada halaman Provider Management/API Key Management, tampilkan untuk setiap provider:

- Provider
- jumlah managed API key
- daftar key yang sudah dimasking
- status enabled/disabled
- key ID/fingerprint jika tersedia
- createdAt jika tersedia

JANGAN pernah menampilkan raw API key setelah key disimpan.

2. ADD API KEY VIA UI

Pastikan admin dapat:

Provider → Add API Key → masukkan API key → Save

Setelah berhasil:

- key tersimpan persistent
- jumlah managed key bertambah
- key langsung tersedia untuk rotation
- UI melakukan refresh data
- tidak perlu restart server.

Saat Save sedang berjalan tampilkan state:
- Saving...

Setelah berhasil:
- kembali ke daftar key
- tampilkan key dalam bentuk masked.

Jika gagal:
- tampilkan error yang jelas
- jangan membuat record setengah jadi
- jangan menampilkan raw secret dalam error.

3. DELETE API KEY VIA UI

Admin dapat memilih key tertentu dan Delete.

Sebelum delete:
- tampilkan confirmation dialog.

Saat delete:
- tampilkan state Working/Deleting.

Setelah berhasil:
- key hilang dari storage
- count berkurang
- key tidak lagi digunakan KeyManager
- UI refresh.

Jika key sedang digunakan oleh rotation:
- rotation harus tetap aman
- jangan sampai request berikutnya crash.

4. ENABLE / DISABLE KEY

Jika arsitektur ApiKeyStore sudah memiliki state enabled/disabled untuk key:

Pastikan UI dapat:

Enable key
Disable key

Disabled key:
- tetap tersimpan
- tetap dihitung sebagai managed key sesuai definisi existing
- tidak boleh dipilih KeyManager untuk request.

Enable kembali:
- langsung dapat digunakan rotation
- tidak membutuhkan restart.

Jika project memang TIDAK memiliki konsep disable per-key:
- jangan membuat sistem baru yang duplikatif.
- pertahankan hanya Provider Enable/Disable dan dokumentasikan bahwa per-key disable belum menjadi bagian schema.

5. COUNT

Pastikan count API key konsisten:

managedKeyCount
=
jumlah managed key yang tersimpan.

Pisahkan:

managedKeyCount
envKeyCount

Jangan:
- mencampur env key dengan managed key
- menghitung duplicate sebagai dua key
- mengurangi count ketika key hanya disabled.

Verifikasi count pada:
- provider list
- provider detail
- dashboard jika ditampilkan
- setelah add
- setelah delete
- setelah restart.

6. DUPLICATE PROTECTION

Saat admin menambahkan key yang sama:

- request ditolak dengan status/error yang sesuai
- tidak membuat duplicate record
- count tidak bertambah
- raw key tidak muncul pada error.

Test juga:
- duplicate managed key
- duplicate terhadap env key jika behavior existing memang mendukung pengecekan tersebut.

7. ROTATION

Gunakan ApiKeyStore + KeyManager existing.

JANGAN membuat KeyManager baru.

Dengan minimal 3 managed key:

A
B
C

Pastikan request menggunakan rotation existing.

Setelah:
- A di-delete
- B di-disable jika supported
- C tetap aktif

KeyManager harus otomatis menyesuaikan tanpa restart.

Pastikan tidak ada:
- stale deleted key
- stale disabled key
- crash karena index rotation
- infinite retry terhadap key yang sudah tidak tersedia.

8. RESTART PERSISTENCE

Lakukan:

ADD KEY
→ verify
→ restart server
→ verify

Pastikan:
- key tetap ada
- raw key tetap tidak ditampilkan
- managedKeyCount tetap benar
- enabled/disabled state tetap benar
- KeyManager dapat menggunakan key setelah restart.

Kemudian:

DELETE KEY
→ restart
→ verify

Pastikan key tidak muncul kembali.

9. SECURITY

Scan seluruh jalur API Key Management.

Raw key TIDAK BOLEH muncul pada:

- frontend HTML
- frontend state
- localStorage
- sessionStorage
- URL
- query parameter
- GET response
- dashboard
- logs
- Usage Logs
- error message
- backup
- console output.

Raw key hanya boleh diterima pada operasi create/update melalui POST body dan kemudian disimpan sesuai mekanisme secret storage existing.

Gunakan:
- masked key
- key ID
- fingerprint

untuk response/listing.

10. BACKUP COMPATIBILITY

Pastikan API key managed:

- TIDAK masuk backup sebagai raw secret.
- Delete tetap permanent.
- Restore backup tidak menciptakan credential palsu.

Jika existing backup memang tidak menyimpan managed credential:
- pertahankan behavior tersebut.

Jangan mengubah backup menjadi penyimpanan raw API key.

11. ADMIN API

Audit endpoint API Key Management existing.

Pastikan tersedia behavior untuk:

- list keys
- add key
- delete key
- count
- enable/disable jika schema mendukung.

Pastikan:
- semua endpoint membutuhkan admin authorization
- GET/list tidak mengembalikan raw key
- DELETE menggunakan key ID, bukan raw key
- POST hanya menerima raw key pada body
- error response tidak membocorkan secret.

Jangan membuat endpoint duplikatif jika endpoint existing sudah tersedia.

12. UI ERROR HANDLING

Pastikan UI menangani:

- duplicate key
- invalid provider
- missing key
- delete failure
- storage failure
- unauthorized admin
- server error
- network error.

Tidak boleh ada UI yang stuck pada:
- Saving...
- Deleting...
- Loading...

Jika request gagal, state harus kembali normal.

13. LOADING & REFRESH

Pastikan:
- initial page load mengambil data terbaru
- add berhasil → refresh
- delete berhasil → refresh
- enable/disable berhasil → refresh
- restart server → data tetap benar.

Jangan menggunakan hardcoded count.

14. TESTING

Tambahkan/pertahankan regression test untuk:

- list managed keys
- add key
- add duplicate key
- delete key
- count after add
- count after delete
- enable/disable key jika supported
- rotation
- delete key during rotation
- restart persistence
- admin authorization
- raw key masking
- raw key tidak muncul di logs
- raw key tidak muncul di backup
- UI error response jika endpoint memiliki testable HTTP contract.

Jalankan:

npm run lint
npm run build
npm test

JANGAN menjalankan atau memicu test/integration test Gorouter.app.

Jika test Gorouter otomatis ikut ditemukan:
- skip/exclude menggunakan mekanisme existing
- jangan mengubah test Gorouter agar terlihat pass
- laporkan jumlah skipped.

15. REGRESSION

Pastikan tidak merusak:

- Provider Management
- Provider Enable/Disable
- Model Registry
- /v1/models
- normal request
- streaming
- Usage Tracking
- Usage Dashboard
- Logs
- Pricing
- Backup/Restore
- API Key rotation.

Jangan melakukan refactor besar.

16. JIKA ADA BUG

WAJIB:

1. Cari root cause.
2. Perbaiki source code.
3. Tambahkan regression test.
4. Jalankan lint.
5. Jalankan build.
6. Jalankan test.
7. Verifikasi ulang flow yang bermasalah.

Jangan hanya melaporkan:
"bug ditemukan".

17. FINAL REPORT

Laporkan:

- Add API key via UI
- Delete API key
- Count managed keys
- envKeyCount
- duplicate protection
- enable/disable key jika supported
- rotation
- restart persistence
- admin authorization
- security masking
- backup compatibility
- endpoint API
- file yang diubah
- test pass/fail/skip
- lint
- build
- masalah yang masih tersisa.

PENTING:

Jangan membuat ApiKeyStore baru.
Jangan membuat KeyManager baru.
Jangan membuat storage baru.
Jangan mengembalikan raw API key ke frontend.
Jangan menyimpan raw API key di backup/log.
Jangan menggunakan Gorouter.app.
Jika ada kesalahan, langsung perbaiki.

```

# Prompt: API Key R
```


Lanjutkan project `nvidia-api`.

JANGAN hanya audit.
Jika menemukan bug atau ketidaksesuaian, LANGSUNG PERBAIKI lalu jalankan test ulang.

FOKUS:
Verifikasi integrasi penuh antara API Key Management → KeyManager → Provider Request → Usage → Logs → Dashboard.

Jangan membuat sistem key baru.
Gunakan ApiKeyStore dan KeyManager existing.

1. MANAGED KEY COUNT

Pastikan jumlah API key pada UI berasal dari ApiKeyStore:

managedKeyCount = jumlah managed API key yang tersimpan.

Pisahkan dengan jelas:
- managedKeyCount
- envKeyCount

Jangan menjumlahkan env key ke managed key.

Pastikan count konsisten pada:
- provider list
- provider detail
- dashboard
- setelah add
- setelah delete
- setelah enable/disable
- setelah restart.

2. REQUEST MENGGUNAKAN ROTATION KEY

Siapkan minimal 3 managed API key untuk provider yang sama.

Contoh:

key A
key B
key C

Lakukan beberapa request nyata/internal melalui provider tersebut.

Pastikan KeyManager melakukan rotation/round-robin menggunakan key yang berbeda sesuai mekanisme existing.

Jangan membuat KeyManager baru setiap request.

Pastikan:
request 1 → key A
request 2 → key B
request 3 → key C
request berikutnya → kembali sesuai rotation state existing.

Jangan tampilkan raw key di output.

3. DELETE KEY

Delete salah satu key yang sedang berada dalam rotation.

Pastikan:
- key benar-benar hilang dari ApiKeyStore
- KeyManager tidak lagi memilih key tersebut
- rotation otomatis menyesuaikan
- request berikutnya menggunakan key yang masih aktif
- tidak terjadi crash
- count berkurang.

4. DISABLE PROVIDER

Disable provider yang memiliki beberapa managed key.

Pastikan:
- semua request baru diblokir
- key tidak digunakan ketika provider disabled
- managed keys tetap tersimpan
- count tetap benar.

Enable kembali:

- rotation kembali aktif
- key tidak perlu ditambahkan ulang.

5. DELETE SEMUA MANAGED KEY

Jika provider tidak memiliki managed key:

Pastikan behavior existing untuk env key tetap benar.

Jangan:
- menghapus env key
- menganggap provider tidak ada
- membuat key otomatis
- menampilkan count managed sebagai env key.

6. DUPLICATE

Tambahkan key yang sama dengan:
- managed key existing
- env key jika applicable

Pastikan duplicate detection sesuai behavior existing.

Jangan sampai duplicate menghasilkan dua managed record.

7. REQUEST → USAGE

Setiap request yang benar-benar diproses harus menghasilkan Usage Record yang benar.

Pastikan Usage Record memiliki:

- provider
- exact model
- status
- HTTP status
- input tokens
- output tokens
- total tokens
- latency
- timestamp
- client/API identifier yang masked.

API key yang digunakan untuk upstream:
- JANGAN disimpan sebagai raw key.
- Jika perlu identifikasi key, gunakan key ID/label/fingerprint yang aman.

8. USAGE COST

Audit jalur:

REQUEST
→ provider
→ exact model
→ upstream usage
→ input/output/total tokens
→ pricing lookup
→ cost calculation
→ usage storage
→ aggregation
→ admin API
→ dashboard.

Pastikan tidak ada jalur yang:
- kehilangan provider
- kehilangan model
- kehilangan token
- menghitung total token dua kali
- menghitung cost dua kali
- memakai pricing model yang salah.

9. PRICING

Untuk setiap usage record yang mempunyai pricing:

inputCost = inputTokens × inputPrice
outputCost = outputTokens × outputPrice

totalCost = inputCost + outputCost

Pastikan pricing menggunakan:
provider + exact model.

Jangan fallback ke harga model lain jika exact pricing tersedia.

Jika pricing tidak tersedia:
- cost harus null/N/A sesuai schema existing
- JANGAN menggunakan $0 sebagai harga sebenarnya.

10. DASHBOARD

Pastikan dashboard menggunakan data aggregation yang sama dengan Usage Store.

Periksa:

- Total Requests
- Successful Requests
- Failed Requests
- Blocked Requests
- Input Tokens
- Output Tokens
- Total Tokens
- Total Cost

Provider breakdown:

Provider
Requests
Tokens
Cost

Model breakdown:

Provider
Model
Requests
Input
Output
Total
Cost

Pastikan angka dashboard dapat direkonsiliasi dengan record detail.

11. RECONCILIATION TEST

Buat dataset test deterministik:

Request A:
input = 100
output = 50

Request B:
input = 200
output = 100

Maka:

total input = 300
total output = 150
total tokens = 450

Cost harus dihitung dari pricing exact model/provider.

Pastikan dashboard:

sum(records) == aggregation == dashboard

Jangan membuat angka hardcoded di production code.

12. REFRESH / RESTART

Setelah beberapa request:

- refresh dashboard
- restart server
- buka dashboard kembali.

Pastikan:
- usage tetap ada
- cost tetap sama
- token tetap sama
- key count tetap benar
- provider state tetap benar
- rotation state tidak corrupt.

13. SECURITY

Scan seluruh jalur baru.

Raw API key TIDAK BOLEH muncul di:

- HTML
- frontend state
- localStorage
- sessionStorage
- URL
- query parameter
- logs
- Usage Record
- Dashboard response
- error message
- backup.

Gunakan masked key / key ID / fingerprint jika identifikasi diperlukan.

14. TESTING

Tambahkan test untuk:

- managed key count
- env key count separation
- key rotation
- delete key saat rotation
- provider disable dengan multiple keys
- enable kembali
- duplicate protection
- request → usage
- usage → pricing
- pricing → cost
- aggregation
- dashboard totals
- restart persistence
- secret leak protection.

Jalankan:

npm run lint
npm run build
npm test

JANGAN menjalankan atau memicu test/integration test Gorouter.app.

Jika test Gorouter otomatis ditemukan:
- skip/exclude secara permanen sesuai mekanisme existing
- jangan mengubah test Gorouter agar terlihat pass
- laporkan jumlah skipped.

15. JIKA ADA BUG

JANGAN hanya menulis "masalah ditemukan".

Langsung:
1. identifikasi root cause
2. perbaiki source code
3. tambahkan regression test
4. jalankan lint
5. jalankan build
6. jalankan test
7. verifikasi ulang flow yang diperbaiki.

Jangan melakukan refactor besar.

16. FINAL REPORT

Laporkan:

- managedKeyCount
- envKeyCount
- rotation
- add/delete
- duplicate
- enable/disable
- request integration
- usage integration
- pricing
- cost calculation
- dashboard reconciliation
- restart persistence
- security audit
- file yang diubah
- test pass/fail/skip
- lint
- build
- masalah yang masih tersisa.

PENTING:
Jangan membuat ApiKeyStore baru.
Jangan membuat KeyManager baru.
Jangan membuat provider/model dummy.
Jangan mengarang token.
Jangan mengarang pricing.
Jangan membocorkan API key.
Jangan menggunakan Gorouter.app.
Jika ada bug, langsung perbaiki.
```
# Prompt: API Key Management — UI Real Verification & Fix
```
Lanjutkan project `nvidia-api`.

JANGAN hanya audit. Jika menemukan masalah, LANGSUNG PERBAIKI lalu test ulang.

FOKUS TAHAP INI:
Verifikasi dan sempurnakan UI Admin untuk API Key Management yang sudah dibuat sebelumnya.

Jangan membuat sistem API key baru.
Gunakan ApiKeyStore + KeyManager existing.

1. UI PROVIDER API KEY

Pada dashboard/admin provider:

Pastikan setiap provider menampilkan:
- Provider name
- Status Active/Disabled
- API Keys count
- daftar managed API key
- masked key
- label/name jika tersedia
- createdAt jika tersedia
- lastUsed jika tersedia

Contoh:

NVIDIA
Active
API Keys: 3

••••••••1111
••••••••2222
••••••••3333

2. ADD API KEY VIA UI

Test flow nyata:

Admin
→ Add API Key
→ pilih provider
→ masukkan API key
→ optional label
→ Save

Pastikan:
- request benar-benar menuju endpoint admin existing
- key tersimpan
- UI otomatis refresh
- count bertambah
- key ditampilkan masked
- raw key tidak pernah muncul setelah save.

Jika API gagal:
- tampilkan error yang jelas
- jangan menghapus data existing
- jangan membuat UI seolah-olah berhasil.

3. DELETE VIA UI

Test:

Delete
→ confirmation
→ API request
→ key terhapus
→ UI refresh
→ count berkurang.

Pastikan delete hanya menghapus managed key yang dipilih.

4. DUPLICATE VIA UI

Masukkan API key yang sama.

Harus:
- ditolak
- UI menampilkan error duplicate
- count tidak berubah
- tidak membuat record kedua.

Jangan membocorkan raw key pada error.

5. ENABLE/DISABLE PROVIDER

Dari UI:

Disable Provider
→ status berubah Disabled
→ request baru tidak menggunakan provider tersebut.

Enable Provider
→ status Active
→ request dapat menggunakan provider kembali.

API key tetap tersimpan ketika provider disabled.

6. KEY COUNT

Pastikan angka API Keys berasal dari data backend sebenarnya.

Jangan:
- hardcode
- menghitung dari data UI lama
- menggunakan env key sebagai managed key.

Bedakan jika sistem memiliki:
- managed API key count
- environment API key count

Jangan mencampurkan keduanya.

7. REFRESH & RESTART

Test:
- refresh browser
- logout/login jika tersedia
- restart server
- buka dashboard kembali.

Pastikan:
- key count tetap benar
- key tetap tersimpan
- provider state tetap benar
- masked key tetap tampil.

8. ROTATION

Setelah minimal 3 key tersedia:

request 1 → key rotation #1
request 2 → key rotation #2
request 3 → key rotation #3

Pastikan UI/API key management tidak membuat KeyManager baru.

Gunakan rotation existing.

9. SECURITY UI

Audit seluruh frontend:

Raw API key TIDAK BOLEH:
- masuk HTML
- masuk response API setelah create
- masuk browser console
- masuk error message
- masuk localStorage
- masuk sessionStorage
- masuk URL/query string.

Pastikan hanya masked representation yang dikirim untuk display.

10. RESPONSIVE UI

Periksa tampilan mobile.

Pastikan:
- Add API Key mudah ditemukan
- form tidak overflow
- table/card responsive
- tombol Delete tetap mudah digunakan
- count terlihat jelas
- confirmation dialog bekerja.

Jangan mengubah desain dashboard secara besar.

11. LOADING & ERROR STATE

Pastikan setiap operasi memiliki state:

Loading
→ Success / Error

Untuk Add/Delete/Refresh.

Cegah double submit/double delete ketika request masih berjalan.

12. TEST

Tambahkan test UI/API integration yang relevan untuk:

- load provider key count
- render masked keys
- add key
- delete key
- duplicate key
- provider disabled
- provider enabled
- refresh persistence
- count synchronization
- raw key tidak muncul di response/UI
- error handling
- loading state.

Jalankan:

npm run lint
npm run build
npm test

JANGAN menjalankan atau memicu test Gorouter.app.

Jika menemukan bug:
LANGSUNG PERBAIKI.
Jangan hanya melaporkan bug.

Jangan mengubah test hanya supaya pass.

13. REGRESSION

Pastikan tetap tidak merusak:

- Provider Management
- Enable/Disable Provider
- Model Registry
- `/v1/models`
- API request
- streaming
- Usage
- Pricing
- Cost calculation
- Dashboard
- Logs
- Backup/Restore
- ApiKeyStore
- KeyManager rotation.

14. FINAL REPORT

Laporkan:

- UI Add API Key
- UI Delete API Key
- API key count
- masked key
- duplicate protection
- provider enable/disable
- rotation
- refresh persistence
- security check
- responsive UI
- file yang diubah
- test pass/fail/skip
- lint
- build
- masalah yang masih tersisa.

PENTING:
Jangan membuat ApiKeyStore baru.
Jangan membuat KeyManager baru.
Jangan menyimpan raw key di frontend.
Jangan menampilkan raw key.
Jangan menggunakan Gorouter.app.
Jika ada bug, langsung perbaiki.


```
# Prompt: Provider API Key Management — Final UI
```

Lanjutkan project `nvidia-api`.

JANGAN hanya audit. Jika menemukan masalah, LANGSUNG PERBAIKI lalu test ulang.

FOKUS:
Selesaikan fitur API Key Management melalui UI admin.

FITUR:

1. PROVIDER API KEY LIST
Pada halaman Provider/API Key Management tampilkan:
- Provider
- jumlah API key
- status provider
- status key aktif/nonaktif
- masked key
- createdAt
- lastUsed jika tersedia

JANGAN pernah menampilkan raw API key setelah disimpan.

2. ADD API KEY

Tambahkan tombol:

+ Add API Key

Form:
- Provider
- API Key
- optional label/name

Saat submit:
- validasi input
- simpan menggunakan ApiKeyStore existing
- jangan menyimpan duplicate key
- jangan menampilkan raw key setelah save
- refresh jumlah API key tanpa restart server.

3. DELETE API KEY

Setiap key memiliki tombol Delete.

Flow:
Delete
→ confirmation
→ hapus key
→ update jumlah key
→ refresh UI

Jangan menghapus provider.

4. KEY COUNT

Tampilkan jumlah API key secara real-time:

Example:
NVIDIA
API Keys: 3

Pastikan count berasal dari storage sebenarnya,
bukan angka hardcoded.

5. ROTATION

Pastikan multiple API key tetap menggunakan KeyManager/rotation existing.

Contoh:

Key A
→ request 1

Key B
→ request 2

Key C
→ request 3

Jangan membuat rotation system kedua.

6. PROVIDER ISOLATION

Key provider A tidak boleh digunakan provider B.

Pastikan setiap request mendapatkan key dari provider yang benar.

7. DISABLE PROVIDER

Jika provider disabled:
- request tidak boleh menggunakan API key provider tersebut
- API key tetap tersimpan
- jumlah key tetap benar

Enable kembali:
- key dapat digunakan lagi.

8. DUPLICATE

Jika API key yang sama ditambahkan dua kali:

→ tolak
→ jangan membuat record kedua.

Pastikan duplicate detection tidak membocorkan raw key ke response/log.

9. SECURITY

Audit seluruh flow:

- raw API key hanya diterima saat create
- storage menggunakan mekanisme secure existing
- logs hanya masked
- API response hanya masked
- dashboard hanya masked
- backup tidak berisi raw key
- error message tidak membocorkan key.

10. UI

Rapikan UI agar:
- responsive mobile
- provider card jelas
- jumlah key terlihat
- tombol Add Key mudah ditemukan
- Delete memiliki confirmation
- masked key mudah dibaca
- loading/error/success state jelas.

Jangan mengubah desain besar dashboard yang sudah ada.

11. REGRESSION

Pastikan tidak merusak:

- Provider Management
- Enable/Disable Provider
- Model Registry
- `/v1/models`
- API request
- streaming
- Usage
- Pricing
- Cost calculation
- Dashboard
- Logs
- Backup/Restore
- existing KeyManager rotation.

12. TEST

Tambahkan/perbaiki test untuk:

- add key
- delete key
- duplicate key
- key count
- masked key
- provider isolation
- multiple key rotation
- disabled provider
- enable provider
- restart persistence
- backup tidak mengandung raw key
- API response tidak mengandung raw key.

Jalankan:

npm run lint
npm run build
npm test

JANGAN menjalankan atau memicu test Gorouter.app.

Jika ada failure:
LANGSUNG cari penyebab dan PERBAIKI.
Jangan mengubah test hanya agar pass.

13. FINAL CHECK

Verifikasi:

Provider NVIDIA
→ 3 API keys
→ UI menampilkan 3
→ add key menjadi 4
→ delete menjadi 3
→ duplicate ditolak
→ rotation tetap bekerja
→ disable provider memblokir request
→ enable provider memulihkan request
→ restart tetap 3 keys.

HASIL AKHIR:

Laporkan:
- file yang diubah
- fitur Add API Key
- Delete API Key
- jumlah API key
- duplicate protection
- rotation
- provider isolation
- security audit
- UI result
- test pass/fail/skip
- lint
- build

JANGAN:
- membuat storage API key baru
- membuat KeyManager kedua
- menyimpan raw key di log
- menampilkan raw key di UI
- menggunakan Gorouter.app
- membuat mock provider

```
# 
```



```
# Prompt: Usage & Pricing Production Hardening
```

Lanjutkan project `nvidia-api`.

JANGAN hanya audit. Jika menemukan masalah, LANGSUNG PERBAIKI lalu test ulang.

Kondisi:
- Pricing → token → costUsd → Usage Store → aggregation → Dashboard sudah terintegrasi.
- Test terakhir: 552 passed, 0 failed, 20 skipped.
- Jangan menjalankan atau memicu test Gorouter.app.

FOKUS:
Production hardening untuk memastikan usage dan cost tetap konsisten setelah restart, perubahan pricing, dan perubahan provider/model.

1. AUDIT RUNTIME

Periksa seluruh runtime flow:

REQUEST
→ PROVIDER
→ MODEL
→ TOKEN USAGE
→ PRICING
→ COST
→ USAGE STORE
→ AGGREGATION
→ ADMIN API
→ DASHBOARD

Jika ada jalur yang masih menghitung cost sendiri atau menggunakan data berbeda:
langsung perbaiki agar menggunakan sumber yang sama.

2. RESTART CONSISTENCY

Test:

server start
→ pricing load
→ request
→ cost benar

restart server
→ request baru
→ cost tetap benar

Pastikan pricing tidak kembali ke default/hardcoded setelah restart.

3. PRICING UPDATE

Test:

Pricing A
→ request
→ cost A

ubah pricing menjadi B
→ request baru
→ cost B

request lama:
→ tetap menggunakan cost yang sudah tercatat.

Jangan menghitung ulang historical cost secara otomatis kecuali memang melalui mekanisme backfill resmi.

4. UNKNOWN PRICING

Untuk model yang belum mempunyai pricing:

costUsd = null

Dashboard harus menampilkan:

N/A

bukan:

$0

Jangan menganggap model tanpa pricing sebagai model gratis.

5. PROVIDER/MODEL ISOLATION

Pastikan pricing exact-match berdasarkan:

provider + exact model ID

Contoh:

provider A + model X
tidak boleh memakai harga:

provider B + model X

atau:

provider A + model Y.

Jika ditemukan fallback pricing yang berpotensi salah:
langsung perbaiki.

6. API KEY ROTATION REGRESSION

Pastikan fitur API Key Management yang sudah dibuat tetap bekerja:

- add API key
- delete API key
- jumlah API key
- key masking
- duplicate protection
- rotation/round-robin existing
- provider isolation

Perubahan API key tidak boleh mengubah:
- usage token
- model
- pricing
- cost calculation.

7. USAGE DASHBOARD

Pastikan dashboard menampilkan konsisten:

Total Requests
Successful
Failed
Blocked

Input Tokens
Output Tokens
Total Tokens

Total Cost

Provider Cost
Model Cost

Pastikan:

Dashboard Total Cost
=
SUM(costUsd yang valid)

Record dengan `costUsd = null`
tidak dihitung sebagai $0.

8. API RESPONSE

Periksa endpoint admin usage/dashboard.

Pastikan nilai:
- token
- request count
- cost
- provider
- model

menggunakan sumber data yang sama.

Jangan membuat endpoint duplikatif.

9. PRECISION

Pastikan perhitungan cost tidak mengalami masalah floating-point.

Gunakan precision yang konsisten.

Test minimal:

input = 1,000,000
output = 500,000
input price = $1/1M
output price = $2/1M

Expected:

input cost = $1
output cost = $1
total = $2

10. DATA INTEGRITY

Pastikan satu request hanya menghasilkan satu usage record.

Jangan terjadi:
- duplicate usage
- duplicate cost
- double aggregation.

Test request success, error, blocked, dan streaming.

11. STREAMING

Pastikan streaming:

- tidak menghasilkan duplicate usage
- tidak kehilangan usage jika upstream memberikan usage
- tetap null jika upstream tidak memberikan usage
- tidak mengganggu response stream.

12. SECURITY

Audit:

- API key tidak masuk log
- Authorization header tidak disimpan
- provider credential tidak masuk usage
- dashboard tidak menampilkan raw secret
- backup tetap tidak berisi credential.

Jika menemukan kebocoran:
langsung perbaiki.

13. TEST

Tambahkan regression test jika diperlukan untuk:

- restart pricing
- pricing update
- historical cost
- unknown pricing
- exact provider/model match
- duplicate usage
- API key rotation
- dashboard total
- provider aggregation
- model aggregation
- streaming usage.

Jalankan:

npm run lint
npm run build
npm test

Jangan menjalankan test Gorouter.app.

Jika ada failure:
langsung cari penyebab sebenarnya dan perbaiki.

Jangan mengubah test hanya agar pass.

14. FINAL VERIFICATION

Verifikasi satu request dari awal sampai akhir:

request
→ provider
→ exact model
→ input token
→ output token
→ total token
→ pricing
→ costUsd
→ usage record
→ provider aggregation
→ model aggregation
→ dashboard.

Pastikan semua angka identik.

HASIL AKHIR:

Laporkan:
- file yang diubah
- bug yang ditemukan
- bug yang diperbaiki
- hasil restart test
- pricing update test
- unknown pricing
- duplicate usage check
- API key regression
- dashboard verification
- security audit
- lint
- build
- test pass/fail/skip.

JANGAN berhenti pada audit.
Jika ada salah → PERBAIKI → TEST ULANG.

```

# Prompt: Final Pricing → Usage → Dashboard
```

Lanjutkan project `nvidia-api`.

JANGAN hanya audit. Jika menemukan masalah, langsung perbaiki dan test ulang.

FOKUS:
Pastikan pricing yang sudah dibuat benar-benar menjadi sumber perhitungan cost untuk seluruh usage dan dashboard.

ALUR WAJIB:

REAL REQUEST
→ provider
→ exact model
→ input tokens
→ output tokens
→ total tokens
→ pricing lookup
→ input cost
→ output cost
→ total cost USD
→ usage storage
→ aggregation
→ admin API
→ dashboard

1. AUDIT SEMUA JALUR COST

Cari seluruh jalur yang menghitung atau menampilkan:
- input tokens
- output tokens
- total tokens
- costUsd
- pricing
- provider/model aggregation

Pastikan tidak ada jalur lama yang masih menghitung cost sendiri.

Semua cost harus menggunakan pricing service/registry yang sama.

Jika ditemukan jalur berbeda atau perhitungan duplikatif:
langsung satukan ke mekanisme pricing existing yang paling benar.

2. EXACT MODEL MATCH

Pricing lookup wajib menggunakan exact:

provider + model ID

Jangan menggunakan:
- partial match
- substring
- nama display model
- model fallback
- harga provider lain.

Jika pricing tidak ditemukan:
costUsd harus tetap null/N/A.

Jangan menganggap model tanpa pricing sebagai free.

3. TOKEN SOURCE

Gunakan token asli dari Usage Store.

Pastikan:

totalTokens = inputTokens + outputTokens

hanya jika kedua nilai tersedia.

Jangan mengestimasi token.

4. COST FORMULA

Gunakan:

inputCost =
(inputTokens / 1,000,000) × inputPricePer1M

outputCost =
(outputTokens / 1,000,000) × outputPricePer1M

totalCost =
inputCost + outputCost

Jangan membulatkan token sebelum perhitungan.

Jangan membulatkan intermediate cost terlalu awal.

5. USAGE RECORD

Pastikan setiap usage record baru menyimpan:

provider
model
inputTokens
outputTokens
totalTokens
costUsd

Jika pricing tidak tersedia:

costUsd = null

Request tetap sukses.

Jangan membuat pricing failure menyebabkan inference gagal.

6. HISTORICAL USAGE

Audit usage lama.

Jika record lama memiliki:
- exact provider
- exact model
- inputTokens
- outputTokens

dan sekarang pricing tersedia:

backfill costUsd secara aman.

Backfill harus idempotent.

Menjalankan backfill dua kali tidak boleh menggandakan atau merusak cost.

Jangan mengubah:
- request ID
- timestamp
- token
- provider
- model
- status.

7. DASHBOARD TOTAL

Pastikan dashboard mengambil data dari Usage Store/aggregation yang sebenarnya.

Tampilkan:

Total Requests
Successful
Failed
Blocked

Input Tokens
Output Tokens
Total Tokens

Total Cost USD

Pastikan:

Dashboard Total Cost
=
SUM(valid costUsd)

Jangan menggunakan hardcoded value.

8. PROVIDER BREAKDOWN

Tampilkan:

Provider
Requests
Input Tokens
Output Tokens
Total Tokens
Cost USD

Pastikan cost provider dihitung hanya dari usage record provider tersebut.

9. MODEL BREAKDOWN

Tampilkan:

Provider
Model
Requests
Input Tokens
Output Tokens
Total Tokens
Cost USD

Model tanpa pricing:

Cost = N/A

Bukan `$0`.

10. LOG DETAIL

Pada detail Usage Log tampilkan:

Provider
Model
Input Tokens
Output Tokens
Total Tokens
Cost USD

Jika cost belum bisa dihitung:

Cost = N/A

11. PRICING UI

Pastikan admin pricing UI menampilkan:

Provider
Model
Input $/1M
Output $/1M
Status

Pastikan perubahan pricing langsung memengaruhi perhitungan usage berikutnya.

Jangan membutuhkan restart server untuk membaca pricing baru jika arsitektur existing memang mendukung runtime update.

Jika cache digunakan:
pastikan cache di-invalidate setelah:
- add pricing
- update pricing
- disable pricing
- delete pricing.

12. PRICE SOURCE

Jangan mengklaim harga builtin sebagai billing aktual provider.

Metadata pricing harus jelas sebagai:

public/list-price estimate

jika memang berasal dari daftar harga publik.

Jangan mengubah harga berdasarkan perkiraan.

Jika provider billing aktual tidak tersedia di environment:
tetap gunakan pricing registry yang tersedia dan tandai sebagai estimasi/list price.

13. PRECISION

Pastikan cost tidak berubah karena floating-point calculation yang tidak aman.

Gunakan mekanisme precision yang sesuai dengan project.

Pastikan contoh:

1,000,000 input
500,000 output
input = $1/1M
output = $2/1M

menghasilkan:

inputCost = $1
outputCost = $1
totalCost = $2

14. CACHE CONSISTENCY

Audit cache pricing.

Pastikan perubahan pricing tidak menggunakan nilai lama.

Test:

pricing A
→ request
→ cost A

update pricing menjadi B
→ request baru
→ cost B

Jangan sampai request kedua masih menggunakan pricing A.

15. API CONSISTENCY

Periksa seluruh endpoint admin usage/dashboard.

Pastikan:
- summary
- provider aggregation
- model aggregation
- records
- logs

semuanya menggunakan sumber cost yang sama.

Tidak boleh ada endpoint yang menghitung cost dengan formula berbeda.

16. TEST

Tambahkan/pertahankan test untuk:

- exact provider/model pricing lookup
- unknown pricing
- input cost
- output cost
- total cost
- null tokens
- historical backfill
- idempotent backfill
- pricing update
- pricing cache invalidation
- provider aggregation
- model aggregation
- dashboard total
- N/A untuk unknown pricing
- precision calculation

Gunakan deterministic test case:

1M input + 0.5M output
input $1
output $2
expected total $2.

17. REGRESSION

Jangan merusak:

- Provider Management
- Enable/Disable Provider
- API Key Management
- Multiple API Keys
- Key rotation
- Model Registry
- `/v1/models`
- `/v1/chat/completions`
- `/v1/responses`
- streaming
- Usage Tracking
- Usage Logs
- Dashboard
- Backup/Restore

Jangan gunakan Gorouter.app.

18. VALIDATION

Jalankan:

npm run lint
npm run build
npm test

Jika ada error akibat perubahan ini:
langsung perbaiki.

Jangan mengubah test hanya agar pass.

Tetap skip test Gorouter sesuai aturan project.

19. FINAL VERIFICATION

Setelah selesai, verifikasi satu jalur lengkap:

request nyata/test request
→ exact model
→ token usage
→ pricing lookup
→ costUsd
→ Usage Store
→ aggregation
→ Dashboard

Pastikan angka pada setiap tahap sama.

HASIL AKHIR:

Laporkan:
- file yang diubah
- bug yang ditemukan
- bug yang diperbaiki
- pricing flow
- cost calculation
- historical backfill
- cache behavior
- dashboard consistency
- test pass/fail/skip
- lint
- build

JANGAN berhenti pada audit.
Jika salah → PERBAIKI → TEST ULANG.

```

# Prompt: Model Pricing & Cost Dashboard
```
Lakukan implementasi dan audit fitur **Model Pricing Management + Cost Calculation** pada project `nvidia-api`.

PENTING:
- Jangan hanya audit.
- Jika menemukan bug atau ketidaksesuaian, langsung perbaiki.
- Jangan membuat provider/model dummy.
- Jangan mengubah API key management yang sudah selesai kecuali diperlukan untuk integrasi.
- Jangan mengarang harga model.
- Jangan mengarang token.
- Gunakan data usage dan pricing yang benar-benar tersimpan di project.

TUJUAN:
Pastikan setiap usage request dapat dihitung menjadi biaya berdasarkan:

input tokens
+ output tokens
+ pricing model
= cost USD

Kemudian hasilnya harus konsisten sampai ke Usage Dashboard.

1. AUDIT PRICING SYSTEM

Audit seluruh jalur:

REQUEST
→ PROVIDER
→ MODEL
→ USAGE TOKENS
→ MODEL PRICING
→ COST CALCULATION
→ USAGE STORE
→ AGGREGATION
→ ADMIN API
→ DASHBOARD

Periksa seluruh file terkait pricing, minimal:
- src/lib/pricing.ts
- src/lib/usage-store.ts
- src/services/provider.ts
- src/routes/admin.ts
- src/admin/dashboard.ts
- src/admin/index.html
- src/admin/styles.css
- model registry
- usage aggregation

Jika ada jalur yang putus, langsung perbaiki.

2. MODEL PRICING REGISTRY

Buat/sempurnakan registry pricing model yang digunakan project.

Setiap model pricing minimal memiliki:

- provider
- exact model ID
- input price per 1M tokens
- output price per 1M tokens
- currency = USD
- active/inactive jika diperlukan
- source/update metadata jika schema existing mendukung

Jangan menggunakan nama model yang berbeda dari model registry.

Exact model ID harus sama dengan model yang muncul di `/v1/models` dan Usage Logs.

3. PRICING CALCULATION

Gunakan formula:

inputCost =
(inputTokens / 1,000,000) * inputPricePer1M

outputCost =
(outputTokens / 1,000,000) * outputPricePer1M

totalCost =
inputCost + outputCost

Pastikan perhitungan menggunakan angka presisi yang aman.

Jangan melakukan pembulatan terlalu awal.

Simpan hasil akhir sesuai precision schema existing.

4. TOKEN VALIDATION

Gunakan token asli dari Usage record.

Jika:

inputTokens = null
atau
outputTokens = null

Jangan mengarang token.

Cost harus:
- null jika cost tidak dapat dihitung secara valid
atau
- mengikuti behavior existing yang sudah ditetapkan project.

Jangan mengubah null menjadi 0 hanya agar dashboard terlihat bagus.

Jika totalTokens tersedia:

totalTokens =
inputTokens + outputTokens

Validasi konsistensinya.

5. HISTORICAL USAGE

Audit record lama.

Jika usage lama mempunyai:
- model
- inputTokens
- outputTokens

tetapi costUsd masih null karena pricing baru tersedia:

Jangan merusak record lama.

Implementasikan backfill hanya jika aman dan sesuai arsitektur existing.

Backfill harus:
- menggunakan exact model ID
- menggunakan pricing yang benar
- tidak mengubah token
- tidak menggandakan record
- tidak mengubah timestamp/request ID
- dapat dijalankan lebih dari sekali tanpa menghasilkan cost ganda.

Jika pricing model memang tidak tersedia:
- cost tetap null
- jangan menggunakan harga perkiraan.

6. ADMIN PRICING UI

Tambahkan halaman/section admin untuk mengelola pricing model.

Minimal:

- Provider
- Model
- Input price / 1M tokens
- Output price / 1M tokens
- Status
- Edit
- Enable/Disable

Jika model sudah terdaftar dari model registry:
gunakan model tersebut.

Jangan membuat model baru dari UI jika model tersebut tidak ada di registry kecuali arsitektur memang membutuhkan pricing entry terpisah.

7. ADD/EDIT PRICING

Admin harus dapat:

- tambah pricing
- edit pricing
- enable pricing
- disable pricing

Validasi:
- provider wajib valid
- model wajib valid
- harga tidak boleh negatif
- harga harus numeric
- input/output price harus valid
- duplicate provider + model harus ditolak atau di-update dengan behavior yang jelas.

8. DELETE PRICING

Jika delete memang diperlukan oleh arsitektur:
- jangan menghapus historical cost yang sudah tersimpan.
- delete hanya pricing configuration.
- usage lama tetap aman.

Jika lebih aman menggunakan disable:
gunakan disable daripada hard delete.

9. USAGE COST

Pastikan setiap usage baru setelah request selesai:

usage record
→ token usage
→ lookup pricing berdasarkan provider + exact model
→ calculate cost
→ simpan costUsd

Pastikan proses cost calculation tidak membuat request API gagal.

Jika pricing tidak ditemukan:
- request tetap berhasil.
- usage tetap tersimpan.
- costUsd = null.
- log/diagnostic boleh mencatat pricing missing tanpa membocorkan secret.

10. USAGE DASHBOARD

Update dashboard agar menampilkan:

TOTAL:
- Total Requests
- Successful Requests
- Failed Requests
- Blocked Requests
- Input Tokens
- Output Tokens
- Total Tokens
- Total Cost USD

PER PROVIDER:
- Provider
- Requests
- Input Tokens
- Output Tokens
- Total Tokens
- Cost USD

PER MODEL:
- Provider
- Model
- Requests
- Input Tokens
- Output Tokens
- Total Tokens
- Cost USD

LOG DETAIL:
- Timestamp
- Provider
- Model
- Status
- Input Tokens
- Output Tokens
- Total Tokens
- Cost USD
- Latency

11. COST CONSISTENCY

Pastikan:

SUM(costUsd setiap usage record)
=
Total Cost Dashboard

Dan:

SUM(inputTokens)
=
Total Input Tokens

SUM(outputTokens)
=
Total Output Tokens

SUM(totalTokens)
=
Total Tokens

Jangan menggunakan angka hardcode di dashboard.

12. UNKNOWN PRICING

Jika model tidak memiliki pricing:

- tampilkan `N/A` atau `—`
- jangan tampilkan `$0`
- jangan menganggap model gratis
- jangan menggunakan harga model lain sebagai fallback.

Ini penting agar dashboard tidak memberikan biaya palsu.

13. API ADMIN

Audit endpoint existing.

Jika sudah ada endpoint pricing:
gunakan endpoint tersebut.

Jika belum ada, tambahkan endpoint yang konsisten, misalnya:

GET    /admin/pricing
POST   /admin/pricing
PUT    /admin/pricing/:id
DELETE /admin/pricing/:id

Sesuaikan dengan routing architecture existing.

Jangan membuat endpoint duplikatif.

14. SECURITY

Pastikan pricing UI/API tidak dapat:
- melihat API key provider
- melihat Authorization header
- melihat secret
- mengubah credential provider

Pricing hanya mengelola metadata harga.

15. TESTING

Tambahkan test untuk:

- pricing lookup
- exact provider + model matching
- input cost calculation
- output cost calculation
- total cost calculation
- null token handling
- unknown pricing
- duplicate pricing
- invalid negative price
- pricing update
- pricing disable
- historical usage
- cost backfill jika dibuat
- dashboard aggregation
- provider aggregation
- model aggregation
- cost total consistency
- request tetap sukses ketika pricing tidak tersedia.

Gunakan contoh deterministik:

input = 1,000,000
output = 500,000
input price = $1
output price = $2

Maka:

input cost = $1
output cost = $1
total cost = $2

16. REGRESSION

Pastikan tidak merusak:

- Provider Management
- Enable/Disable Provider
- API Key Management
- Multiple API Keys
- Key rotation
- Model Registry
- `/v1/models`
- `/v1/chat/completions`
- `/v1/responses`
- streaming
- Usage Tracking
- Usage Logs
- Usage Dashboard
- Backup/Restore

Jangan menjalankan atau menggunakan Gorouter sebagai fallback.

17. VALIDATION

Jalankan:

npm run lint
npm run build
npm test

Jika menemukan failure:

JANGAN hanya melaporkan.

Cari penyebabnya dan langsung perbaiki jika memang disebabkan perubahan ini.

Jangan mengubah test hanya agar test menjadi hijau.

Setelah perbaikan, jalankan ulang test yang relevan dan full test suite.

18. HASIL AKHIR

Laporkan:

- file yang diubah
- pricing registry
- pricing API
- pricing UI
- formula cost
- historical usage handling
- unknown pricing behavior
- dashboard cost
- test baru
- lint
- build
- total test pass/fail/skip
- bug yang ditemukan dan diperbaiki
- masalah yang benar-benar masih tersisa

PENTING:
Jangan berhenti pada audit.
Jika ada bug → langsung perbaiki → test ulang → lanjutkan sampai pipeline pricing konsisten.

TARGET AKHIR:

REAL REQUEST
→ REAL PROVIDER
→ REAL MODEL
→ REAL INPUT TOKENS
→ REAL OUTPUT TOKENS
→ MODEL PRICING
→ REAL COST USD
→ USAGE STORE
→ AGGREGATION
→ ADMIN DASHBOARD


```

# Prompt 14 — Final Usage Cost Hardening & Fix
```

Lanjutkan project `nvidia-api` dari hasil audit Usage terakhir.

STATUS TERAKHIR:
- 527 tests passed
- 0 failed
- 20 skipped
- Usage audit sudah mencakup extraction, totals, mapping, pricing, split cost, free/unknown/null, streaming, aggregation, dashboard consistency, cross-provider separation, precision, round-trip storage.
- Jangan mengulang pekerjaan yang sudah terbukti benar.

PENTING:
JANGAN hanya membuat laporan.
Audit → temukan root cause → LANGSUNG PERBAIKI jika aman → tambahkan regression test → jalankan test ulang.

==================================================
1. ERROR FALLBACK ATTRIBUTION
==================================================

Temuan terakhir:

Saat SEMUA provider gagal, error record saat ini tetap menggunakan behavior existing:
- satu request menghasilkan satu request record
- fallback dapat memiliki attempt/provider error information
- perubahan attribution tidak boleh mengubah semantic request count.

Audit source code yang menangani:
- provider attempt
- fallback
- error recording
- usage recording
- routingLog
- attribution.

Tujuan:

Pastikan satu request tidak berubah menjadi multiple request count hanya karena mencoba beberapa provider.

Namun tetap pertahankan informasi attempt/error yang diperlukan.

Jika ada bug nyata:
- PERBAIKI LANGSUNG.

Pastikan:

1 logical API request
→ 1 request-level usage count

Sedangkan provider attempts/error dapat disimpan sebagai metadata/attempt information sesuai schema existing.

Test wajib:

- provider pertama gagal → provider kedua sukses
- provider pertama gagal → provider kedua gagal
- semua provider gagal
- satu provider tanpa fallback
- multiple attempts tidak menggandakan request count
- successful fallback tetap tercatat success
- all-provider failure tetap tercatat error
- provider/model attribution tidak salah.

JANGAN mengubah semantic request count hanya untuk membuat test baru lulus.

==================================================
2. HISTORICAL COST DATA
==================================================

Temuan:

Historical record tertentu hanya memiliki:

costUsd

tetapi tidak memiliki data yang cukup untuk merekonstruksi:

inputCost
outputCost

Jangan mengarang data.

Audit migration/backfill yang sudah dibuat.

Pastikan:

Jika historical record memiliki:
- exact model
- provider
- input tokens
- output tokens
- pricing yang valid

→ boleh dihitung ulang secara deterministic.

Jika historical record hanya memiliki:
- costUsd

atau data pricing/token tidak lengkap:

→ PERTAHANKAN `costUsd` existing.
→ Jangan membuat inputCost/outputCost palsu.
→ split cost tetap `null`/N/A.
→ jangan mengubah total cost.

Pastikan migration:

- idempotent
- tidak menggandakan cost
- tidak mengubah timestamp
- tidak mengubah provider
- tidak mengubah model
- tidak mengubah token
- tidak mengubah historical total cost.

Tambahkan regression test:

- historical cost-only record
- historical full-token record
- historical incomplete record
- migration dijalankan dua kali
- total cost tetap sama.

Jika implementation saat ini sudah benar:
JANGAN melakukan perubahan yang tidak perlu.

==================================================
3. COST CALCULATION SOURCE OF TRUTH
==================================================

Audit seluruh jalur:

provider response
→ normalized usage
→ model/provider pricing
→ inputCost
→ outputCost
→ totalCost
→ usage storage
→ aggregation
→ admin API
→ dashboard.

Pastikan TIDAK ada jalur kedua yang menghitung cost dengan formula berbeda.

Formula:

inputCost =
(inputTokens / pricingUnit) × inputPrice

outputCost =
(outputTokens / pricingUnit) × outputPrice

totalCost =
inputCost + outputCost

Gunakan precision internal yang cukup.

Jangan menghitung menggunakan angka yang sudah diformat UI.

Jangan menggunakan:
- `$0.00`
- string currency
- rounded display value

sebagai source perhitungan.

==================================================
4. PRICING SEMANTIC
==================================================

Pastikan tiga kondisi tetap berbeda:

KNOWN:
pricing tersedia
→ cost dihitung

FREE:
model memang free
→ cost = 0

UNKNOWN:
pricing tidak diketahui
→ cost = null/N/A

JANGAN:

UNKNOWN → $0

Jangan fallback ke harga model lain.

Pricing lookup harus mempertimbangkan:

provider + exact model

bukan hanya model ID jika model yang sama dapat berada di provider berbeda.

==================================================
5. TOKEN ACCOUNTING
==================================================

Audit seluruh endpoint:

- `/v1/chat/completions`
- `/v1/responses`
- streaming
- provider fallback.

Usage harus berasal dari upstream/provider response.

Jika tersedia:

inputTokens
outputTokens
totalTokens

Pastikan:

totalTokens = inputTokens + outputTokens

Jika upstream hanya memberikan sebagian:
- jangan mengarang nilai
- simpan null sesuai schema.

Jika streaming mengirim:

usage: null

jangan membuat estimasi.

==================================================
6. DASHBOARD SOURCE OF TRUTH
==================================================

Dashboard TIDAK boleh menghitung cost/token sendiri dari raw records dengan formula berbeda.

Backend usage aggregation adalah source of truth.

Pastikan:

Usage records
=
provider aggregation
=
model aggregation
=
dashboard totals

Untuk cost:

known cost → dijumlahkan

unknown/null cost → jangan diam-diam dianggap $0 jika semantic project membedakannya sebagai UNKNOWN.

Pastikan dashboard menampilkan status pricing/cost dengan jelas jika diperlukan:

- Known
- Free
- N/A

==================================================
7. API KEY / PROVIDER SEPARATION
==================================================

Pastikan usage tetap dapat dipisahkan berdasarkan:

- provider
- exact model
- client/API key identifier jika tersedia.

Jangan menyimpan raw API key.

Pastikan:

provider A + model X

tidak tercampur dengan:

provider B + model X.

==================================================
8. STREAMING
==================================================

Regression test:

stream request
→ response tetap lancar
→ stream selesai
→ usage record dibuat
→ usage provider digunakan jika tersedia
→ cost dihitung jika token + pricing tersedia.

Jika usage upstream null:

token = null
cost = null/N/A

Jangan membuat stream gagal hanya karena usage null.

==================================================
9. ERROR / BLOCKED / SUCCESS
==================================================

Pastikan usage membedakan:

SUCCESS
ERROR
BLOCKED

SUCCESS:
- request berhasil
- HTTP status aktual
- usage jika tersedia
- cost jika dapat dihitung

ERROR:
- provider/upstream gagal
- HTTP status aktual jika tersedia
- jangan menganggap success
- jangan menghitung cost tanpa usage valid

BLOCKED:
- request diblokir sebelum upstream
- tidak boleh dihitung sebagai successful upstream request.

==================================================
10. REAL REQUEST VALIDATION
==================================================

Jangan memaksakan real paid-provider validation jika environment belum memiliki credential yang valid.

Jika credential produksi tidak tersedia:
- jangan membuat credential palsu
- jangan mengubah source code untuk bypass authorization
- jangan menganggap validasi tersebut gagal sebagai bug internal.

Namun seluruh internal calculation/storage/aggregation harus tetap dapat divalidasi dengan test yang deterministic dan data provider yang memang sudah tersedia di project.

==================================================
11. AUTO-FIX RULE
==================================================

Jika menemukan bug:

1. Identifikasi root cause.
2. Perbaiki source code.
3. Tambahkan regression test.
4. Jalankan test terkait.
5. Jalankan lint.
6. Jalankan build.
7. Audit ulang jalur yang berubah.

JANGAN berhenti pada:
"masalah ditemukan".

Target:
"masalah ditemukan → diperbaiki → diverifikasi".

==================================================
12. REGRESSION
==================================================

Jangan merusak:

- Provider Management
- API Key Management
- Enable/Disable Provider
- Model Registry
- `/v1/models`
- `/v1/chat/completions`
- `/v1/responses`
- streaming
- Usage Tracking
- Usage Logs
- Pricing
- Usage Dashboard
- Backup/Restore.

Jangan melakukan refactor besar.

==================================================
13. TEST
==================================================

Tambahkan/perbaiki test untuk:

- fallback success
- fallback all failed
- request count tidak double count
- provider attempt attribution
- historical cost-only
- historical full-cost reconstruction
- idempotent migration
- known pricing
- free pricing
- unknown pricing
- provider/model pricing separation
- exact token calculation
- null token
- streaming usage null
- streaming usage tersedia
- success/error/blocked
- dashboard aggregation
- cost aggregation
- round-trip storage.

Jalankan:

npm run lint
npm run build
npm test

JANGAN menjalankan atau memicu test/integration test Gorouter.app.

Jika test Gorouter otomatis ditemukan:
- skip/exclude
- jangan mengubah test agar lulus
- laporkan jumlah skip.

==================================================
14. FINAL VERIFICATION
==================================================

Setelah semua selesai, lakukan final audit terhadap jalur:

REQUEST
↓
PROVIDER
↓
MODEL
↓
UPSTREAM USAGE
↓
NORMALIZED USAGE
↓
PRICING
↓
INPUT COST
↓
OUTPUT COST
↓
TOTAL COST
↓
USAGE STORE
↓
AGGREGATION
↓
ADMIN API
↓
DASHBOARD

Pastikan tidak ada jalur yang:
- kehilangan token
- menggandakan request
- menggandakan cost
- mencampur provider
- mencampur model
- mengubah UNKNOWN menjadi $0
- menghitung cost dari angka UI
- mengarang token.

==================================================
15. HASIL AKHIR
==================================================

Laporkan:

- bug yang ditemukan
- root cause
- source file yang diperbaiki
- perubahan yang dilakukan
- test baru
- hasil fallback
- hasil historical cost handling
- hasil pricing validation
- hasil token validation
- hasil dashboard consistency
- lint
- build
- test
- jumlah passed/failed/skipped
- masalah yang benar-benar masih membutuhkan credential/data eksternal.

PENTING:

Jangan hanya audit.
Jika ada kesalahan di source code dan aman diperbaiki, LANGSUNG PERBAIKI.

Jangan mengubah behavior yang memang sudah benar hanya untuk menghilangkan "masalah tersisa".

Audit → Fix → Test → Verify.

```

# Prompt 13 — Fix Remaining Usage Cost & Historical Data
```
Lanjutkan dari hasil audit Usage terakhir pada project `nvidia-api`.

HASIL TERAKHIR:
- 518 tests passed
- 0 failed
- 20 skipped
- Usage audit sudah mencakup extraction, totals, mapping, pricing, split cost, free/unknown/null, streaming, aggregation, dashboard consistency, cross-provider separation, precision, dan round-trip storage.

JANGAN hanya membuat laporan.
Jika ditemukan masalah yang aman diperbaiki, LANGSUNG PERBAIKI SOURCE CODE + TEST.

==================================================
1. FIX ERROR FALLBACK ATTRIBUTION
==================================================

Temuan:

Saat semua provider gagal, error fallback attribution saat ini dapat didistribusikan ke provider percobaan terakhir.

Audit root cause pada jalur fallback/error handling.

Tujuan:

Jika request mencoba beberapa provider dan semuanya gagal:
- jangan mengubah attribution provider secara sembarangan
- jangan membuat provider terakhir terlihat sebagai provider yang berhasil
- jangan mengubah semantic request count existing tanpa alasan kuat
- provider/model pada usage harus merepresentasikan request/attempt sebenarnya sesuai arsitektur existing.

Pastikan:
- success attribution tetap benar
- upstream error attribution tetap benar
- blocked attribution tetap benar
- multi-provider fallback tetap dapat dibedakan
- tidak terjadi double counting.

Tambahkan regression test untuk:
- provider pertama gagal
- provider kedua gagal
- seluruh provider gagal
- provider fallback berhasil
- provider fallback semuanya gagal.

Jangan mengubah behavior yang sudah benar hanya demi membuat test baru lulus.

==================================================
2. HISTORICAL COST RECORD
==================================================

Temuan:

Sebagian historical record memiliki:
- `costUsd` numeric
- tetapi belum memiliki split:
  - inputCost
  - outputCost

Jangan merusak historical data.

Audit schema dan storage usage.

Tentukan apakah record lama memiliki informasi yang cukup untuk menghitung ulang:

input tokens
output tokens
provider
exact model
pricing

Jika SEMUA data tersedia:
→ lakukan safe backfill.

Jika data tidak cukup:
→ jangan mengarang split cost.

Untuk historical record yang hanya memiliki:
`costUsd`

dan tidak memiliki informasi token/pricing yang cukup:

- pertahankan `costUsd` existing
- jangan mengubah nilainya
- jangan membuat inputCost/outputCost palsu
- tampilkan split sebagai null/N/A jika diperlukan.

Tujuan utama:
Historical total cost tidak boleh berubah hanya karena migration.

Jika membuat migration:
- harus idempotent
- aman dijalankan berkali-kali
- tidak menggandakan cost
- tidak mengubah timestamp
- tidak mengubah provider
- tidak mengubah model
- tidak mengubah token asli.

Tambahkan test:
- old record with total cost only
- old record with full token/pricing data
- old record with insufficient data
- repeated migration
- migration preserves existing total cost.

==================================================
3. END-TO-END COST VALIDATION
==================================================

Buat validasi end-to-end menggunakan jalur request nyata/internal yang sudah tersedia.

Validasi:

request
→ provider
→ model
→ usage
→ pricing
→ input cost
→ output cost
→ total cost
→ storage
→ aggregation
→ admin API
→ dashboard.

Jangan menggunakan mock provider untuk menggantikan request nyata jika environment sudah menyediakan data nyata.

Gunakan data request yang aman.

Validasi secara matematis:

inputCost =
(inputTokens / pricingUnit) × inputPrice

outputCost =
(outputTokens / pricingUnit) × outputPrice

totalCost =
inputCost + outputCost

Bandingkan:

calculated total
vs
stored total
vs
aggregated dashboard total.

Harus konsisten dalam tolerance precision yang wajar.

Jangan menggunakan angka hasil formatting UI untuk perhitungan.

==================================================
4. PRICING STATUS
==================================================

Pastikan tiga kondisi tetap dibedakan:

KNOWN
→ pricing tersedia
→ cost dapat dihitung

FREE
→ model memang dikonfigurasi free
→ cost = 0

UNKNOWN
→ pricing tidak diketahui
→ cost = null/N/A

JANGAN:

unknown → $0

Jangan membuat fallback pricing dari model/provider lain.

==================================================
5. TOKEN SOURCE
==================================================

Pastikan seluruh cost tetap bergantung pada token usage asli.

Prioritas:

provider usage response
→ normalized usage
→ pricing
→ cost.

Jangan:
- menghitung token dari panjang prompt
- mengestimasi token
- mengarang token ketika upstream tidak memberikan usage.

Jika token null:
→ cost null/N/A kecuali model benar-benar free dan behavior existing memang menetapkan cost $0.

==================================================
6. STREAMING REGRESSION

Pastikan perubahan tidak merusak streaming.

Test:

stream success
→ stream selesai
→ usage jika tersedia
→ pricing
→ cost jika token + pricing tersedia
→ satu usage record.

Jika streaming usage null:
→ token null
→ cost null/N/A
→ stream tetap sukses.

Jangan membuat streaming gagal hanya karena usage tidak tersedia.

==================================================
7. DASHBOARD FINAL CONSISTENCY

Pastikan dashboard tidak menghitung cost sendiri.

Backend/storage menjadi source of truth.

Dashboard harus menampilkan hasil aggregation backend.

Validasi:

sum(record.totalCost)
=
provider aggregation total
=
model aggregation total
=
dashboard total cost

Untuk record yang cost null:
- jangan dihitung sebagai $0 secara diam-diam jika status pricing UNKNOWN.
- aggregation harus mengikuti semantic existing yang sudah ditetapkan.

==================================================
8. PROVIDER / MODEL SEPARATION

Pastikan:

provider A + model X
dan
provider B + model X

tetap menggunakan pricing context masing-masing.

Jangan lookup pricing berdasarkan model ID saja jika model ID dapat muncul pada provider berbeda.

Pricing key harus mempertimbangkan provider + exact model.

==================================================
9. API KEY / CLIENT USAGE

Pastikan usage attribution berdasarkan API key/client tetap aman.

Gunakan identifier yang sudah dimasking.

Jangan menyimpan:
- raw API key
- Authorization header
- provider credential.

Pastikan cost aggregation per client jika fitur tersebut sudah tersedia tidak mencampur client.

==================================================
10. TEST

Tambahkan/perbaiki test untuk:

- all providers fail attribution
- fallback provider success
- fallback providers all fail
- historical total cost only
- historical record full token data
- historical record insufficient data
- idempotent cost migration
- known pricing
- free pricing
- unknown pricing
- null token
- streaming usage
- streaming null usage
- provider/model pricing separation
- exact cost calculation
- dashboard total cost
- provider total cost
- model total cost
- round-trip storage.

Jalankan:

npm run lint
npm run build
npm test

PENTING:
Jangan menjalankan atau memicu test/integration test Gorouter.app.

Jika test suite otomatis memuat test Gorouter:
- skip/exclude test tersebut.
- jangan mengubah test Gorouter agar terlihat lulus.

==================================================
11. AUTO-FIX
==================================================

Jika test menemukan bug:

1. Cari root cause.
2. Perbaiki source code.
3. Tambahkan regression test.
4. Jalankan ulang test.
5. Audit ulang jalur yang terkena perubahan.

Jangan berhenti pada laporan.

==================================================
12. REGRESSION

Pastikan tetap tidak merusak:

- Provider Management
- API Key Management
- Provider Enable/Disable
- Model Registry
- `/v1/models`
- `/v1/chat/completions`
- `/v1/responses`
- streaming
- Usage Tracking
- Usage Logs
- Pricing
- Usage Dashboard
- Backup/Restore.

Jangan melakukan refactor besar jika tidak diperlukan.

==================================================
13. FINAL REPORT

Setelah selesai laporkan:

- 3 masalah awal
- root cause masing-masing
- file yang diperbaiki
- perubahan source code
- migration/backfill yang dilakukan
- historical records yang berhasil diperbaiki
- records yang tetap N/A dan alasannya
- hasil cost calculation
- hasil dashboard consistency
- hasil fallback attribution
- test tambahan
- npm run lint
- npm run build
- npm test
- jumlah passed/failed/skipped
- masalah yang masih tersisa.

TARGET AKHIR:

Tidak boleh ada bug yang diketahui dan aman diperbaiki tetapi hanya dilaporkan.

Audit → Fix → Test → Verify.


```

# Prompt 12 — Full Usage Token → Pricing → Cost Audit & Fix
```


Lakukan FULL AUDIT + FIX pada seluruh jalur Usage di project `nvidia-api`.

TUJUAN UTAMA:

Pastikan seluruh request API memiliki alur data yang benar:

MODEL REQUEST
→ PROVIDER
→ INPUT/PROMPT TOKENS
→ OUTPUT/COMPLETION TOKENS
→ TOTAL TOKENS
→ MODEL PRICING
→ INPUT COST
→ OUTPUT COST
→ TOTAL COST
→ USAGE STORAGE
→ AGGREGATION
→ ADMIN DASHBOARD

PENTING:
Jika menemukan bug, ketidakkonsistenan, perhitungan salah, data null yang seharusnya dapat dihitung, atau dashboard membaca sumber data yang salah:
→ LANGSUNG PERBAIKI SOURCE CODE.
Jangan hanya membuat laporan audit.

Jangan menunggu prompt berikutnya untuk memperbaiki bug yang ditemukan.

==================================================
1. AUDIT SELURUH JALUR USAGE
==================================================

Audit semua source yang berhubungan dengan:

- src/lib/usage-store.ts
- src/lib/pricing.ts
- src/services/stream-usage.ts
- src/services/provider.ts
- src/lib/model-registry.ts
- src/routes/admin.ts
- src/admin/dashboard.ts
- src/admin/index.html
- src/admin/styles.css
- seluruh service/request handler yang mencatat usage
- seluruh test usage/pricing/cost/dashboard

Cari semua tempat yang:
- membuat usage record
- mengubah usage record
- menghitung token
- menghitung cost
- mengambil pricing
- melakukan aggregation
- mengirim data ke dashboard

Jangan hanya mengikuti satu jalur request.
Audit semua jalur request yang ada.

==================================================
2. TOKEN SOURCE OF TRUTH
==================================================

Pastikan token selalu berasal dari usage asli provider/API jika tersedia.

Field yang harus konsisten:

inputTokens / promptTokens
outputTokens / completionTokens
totalTokens

Validasi:

totalTokens = inputTokens + outputTokens

Jika upstream memberikan total_tokens:
- bandingkan dengan hasil penjumlahan.
- gunakan nilai upstream jika memang menjadi source of truth project.
- jika berbeda, jangan diam-diam menyembunyikan discrepancy.
- perbaiki mapping yang menyebabkan perbedaan.

Jika provider tidak memberikan usage:
- jangan mengarang token.
- jangan melakukan estimasi kecuali project memang sudah memiliki mekanisme resmi yang sengaja digunakan.
- simpan null sesuai schema existing.

Pastikan mapping usage konsisten untuk:
- NVIDIA
- TokenHarbor
- provider lain yang sudah terdaftar
- non-streaming
- streaming
- success
- error
- blocked

==================================================
3. PRICING AUDIT
==================================================

Audit `src/lib/pricing.ts` dan seluruh sumber pricing.

Pastikan pricing berdasarkan:

provider + exact model ID

Jangan hanya berdasarkan nama model yang tampil di UI.

Untuk setiap model yang memiliki pricing, pastikan tersedia:

- input price
- output price
- unit pricing
- currency jika memang tersedia
- source/version pricing jika struktur project mendukungnya

Jangan menggunakan harga palsu.

Jangan mengasumsikan semua model memiliki harga yang sama.

Jangan menggunakan harga model lain sebagai fallback secara diam-diam.

Jika model benar-benar gratis:
- pricing harus secara eksplisit menunjukkan free/zero sesuai metadata pricing yang benar.

Jika harga model tidak diketahui:
- cost harus `null` / `N/A` sesuai schema existing.
- JANGAN menganggap unknown = $0.

==================================================
4. COST CALCULATION
==================================================

Audit dan perbaiki perhitungan cost.

Gunakan formula yang sesuai dengan unit pricing.

Jika pricing menggunakan USD per 1M token:

inputCost =
(inputTokens / 1_000_000) * inputPrice

outputCost =
(outputTokens / 1_000_000) * outputPrice

totalCost =
inputCost + outputCost

Pastikan:
- tidak integer division
- tidak salah unit
- tidak membulatkan token sebelum perhitungan
- tidak double count
- tidak menghitung total token dengan harga input saja
- output token menggunakan output price
- input token menggunakan input price

Gunakan decimal/number handling yang cukup aman untuk nilai cost kecil.

Jangan membulatkan terlalu awal.

Jika input/output token null:
- jangan membuat cost palsu.

Jika pricing null:
- cost tetap null/N/A.

==================================================
5. PER-REQUEST COST
==================================================

Setiap usage record yang dapat dihitung harus memiliki cost yang konsisten.

Minimal:

inputTokens
outputTokens
totalTokens
inputCost
outputCost
totalCost

Pastikan cost dihitung berdasarkan:

exact provider
+
exact model
+
actual token usage
+
pricing yang benar.

Jangan menghitung cost hanya di frontend.

Backend/storage harus menjadi source of truth.

Frontend hanya menampilkan hasil yang sudah dihitung.

==================================================
6. HISTORICAL USAGE RECORDS
==================================================

Audit record lama.

Temuan sebelumnya menunjukkan:
- ada production records dengan `costUsd: null`
- sebagian record lama memiliki costUsd float
- sebagian model belum memiliki pricing registry.

Periksa seluruh record existing.

Jika token dan pricing tersedia:
- lakukan safe recalculation cost.
- jangan mengubah token asli.
- jangan mengubah provider/model asli.
- jangan mengubah timestamp asli.

Jika pricing tidak tersedia:
- tetap null/N/A.
- jangan membuat angka.

Jika record lama memiliki cost yang salah:
- perbaiki berdasarkan token + pricing yang benar.

Pastikan migration/recalculation aman dan idempotent.

Menjalankan recalculation dua kali tidak boleh menggandakan cost.

==================================================
7. STREAMING USAGE
==================================================

Audit jalur streaming secara khusus.

Pastikan:

stream request
→ usage collection
→ final usage
→ token extraction
→ pricing
→ cost
→ usage record

Jika final streaming chunk memberikan usage:
- gunakan usage tersebut.

Jika final chunk `usage: null`:
- jangan mengarang token.
- cost tetap null jika token tidak tersedia.

Jangan sampai streaming menghasilkan:
- duplicate usage record
- duplicate cost
- double request count
- cost dihitung dua kali.

==================================================
8. ERROR & BLOCKED REQUEST
==================================================

Pastikan request berikut tidak salah dihitung sebagai successful paid usage:

- invalid model
- provider disabled
- blocked request
- authentication failure
- validation failure
- upstream error sebelum usage tersedia

Audit apakah error request yang memiliki sebagian usage memang harus dihitung sesuai arsitektur existing.

Jangan menganggap semua HTTP request = billable success.

Pastikan status:

success
error
blocked

tetap dibedakan.

==================================================
9. PROVIDER + MODEL CONSISTENCY
==================================================

Untuk setiap usage record:

provider harus berasal dari provider yang benar-benar menangani request.

model harus exact model ID yang benar-benar digunakan.

Pricing lookup harus menggunakan provider + exact model.

Jangan sampai:

request model A
→ log model A
→ pricing model B.

Tambahkan regression test untuk kasus ini.

==================================================
10. DASHBOARD SOURCE OF TRUTH
==================================================

Audit dashboard.

Dashboard TIDAK BOLEH menghitung ulang cost dengan formula berbeda dari backend.

Dashboard harus mengambil hasil aggregation dari usage service/backend.

Periksa:

- Total Requests
- Successful Requests
- Failed Requests
- Blocked Requests
- Input Tokens
- Output Tokens
- Total Tokens
- Total Cost
- Cost per Provider
- Cost per Model
- Cost per API Key/client jika tersedia
- Average latency

Pastikan:

Dashboard Total Cost
=
sum(valid usage record totalCost)

Jangan:

Dashboard menghitung harga sendiri dari token dengan formula berbeda.

==================================================
11. PROVIDER BREAKDOWN
==================================================

Per provider tampilkan:

Provider
Requests
Success
Error
Blocked
Input Tokens
Output Tokens
Total Tokens
Total Cost

Pastikan aggregation provider menggunakan exact provider ID.

==================================================
12. MODEL BREAKDOWN
==================================================

Per model tampilkan:

Provider
Model
Requests
Success
Error
Blocked
Input Tokens
Output Tokens
Total Tokens
Input Cost
Output Cost
Total Cost

Pastikan model yang sama dari provider berbeda tidak tercampur.

Contoh:

providerA/model-x
dan
providerB/model-x

harus menjadi dua pricing context berbeda.

==================================================
13. UNKNOWN PRICING
==================================================

Jangan menyembunyikan model yang belum memiliki pricing.

Dashboard harus dapat membedakan:

Known pricing
→ cost tersedia

Unknown pricing
→ cost N/A/null

Free model
→ cost $0 jika benar-benar dikonfigurasi sebagai free.

Jangan:

unknown pricing = $0

karena itu membuat total biaya menjadi salah.

Jika perlu tambahkan field:

pricingStatus:
- known
- free
- unknown

Gunakan pola schema yang paling kompatibel dengan project existing.

==================================================
14. API RESPONSE
==================================================

Audit endpoint:

/admin/usage
/admin/usage/providers
/admin/usage/models
/admin/usage/records
/admin/logs

Pastikan response memiliki data cost yang konsisten.

Jangan membuat endpoint duplikatif.

Pertahankan kompatibilitas endpoint existing.

==================================================
15. PRECISION & ROUNDING
==================================================

Audit seluruh pembulatan cost.

Jangan membulatkan:

input token
output token
total token

sebelum calculation.

Cost boleh diformat untuk UI, misalnya:

$0.00012345

tetapi nilai internal harus mempertahankan precision yang cukup.

Pastikan aggregation menggunakan nilai internal, bukan angka yang sudah diformat UI.

==================================================
16. TEST WAJIB
==================================================

Tambahkan/perbaiki test untuk:

1. token extraction success
2. total token calculation
3. provider/model mapping
4. pricing lookup
5. input cost calculation
6. output cost calculation
7. total cost calculation
8. free model
9. unknown pricing
10. null token usage
11. streaming usage
12. streaming usage null
13. duplicate streaming protection
14. provider aggregation
15. model aggregation
16. total cost aggregation
17. historical cost recalculation
18. idempotent recalculation
19. blocked request
20. invalid model
21. upstream error
22. dashboard cost consistency
23. provider A/model X vs provider B/model X
24. precision calculation

==================================================
17. REAL DATA VALIDATION
==================================================

Gunakan data usage existing untuk memverifikasi hasil.

Jangan membuat fake production records hanya untuk membuat test hijau.

Ambil beberapa record existing dan hitung manual:

input tokens
+
output tokens
=
total tokens

Kemudian:

input tokens × input price
+
output tokens × output price
=
total cost

Bandingkan dengan hasil backend.

Jika berbeda:
→ cari sumber bug
→ perbaiki source code
→ jalankan test ulang.

==================================================
18. SECURITY
==================================================

Pastikan audit/fix ini tidak menyebabkan:

- API key masuk log
- Authorization header masuk usage
- provider secret masuk dashboard
- raw credential masuk backup
- prompt/completion sensitif ditampilkan tanpa kebutuhan.

==================================================
19. REGRESSION
==================================================

Jangan merusak:

- Provider Management
- Enable/Disable Provider
- API Key Management
- Model Registry
- `/v1/models`
- `/v1/chat/completions`
- `/v1/responses`
- streaming
- Usage Tracking
- Usage Logs
- Pricing
- Dashboard
- Backup/Restore

Jangan melakukan refactor besar jika tidak diperlukan.

==================================================
20. TEST COMMAND
==================================================

Jalankan:

npm run lint
npm run build
npm test

PENTING:

Jangan menjalankan atau memicu test/integration test Gorouter.app.

Jika test suite otomatis menemukan test Gorouter:
- skip/exclude test tersebut.
- jangan mengubah test Gorouter hanya agar lulus.
- laporkan jumlah test yang di-skip.

NVIDIA dan TokenHarbor boleh dites jika diperlukan untuk validasi jalur usage.

==================================================
21. AUTO-FIX RULE
==================================================

INI WAJIB.

Jika menemukan masalah:

JANGAN hanya menulis:

"Found issue: ..."

Tetapi:

1. identifikasi root cause
2. perbaiki source code
3. tambahkan/perbaiki regression test
4. jalankan test
5. jika test gagal, perbaiki lagi
6. audit ulang jalur yang terkena perubahan.

Jangan berhenti pada audit report jika bug dapat diperbaiki secara aman.

==================================================
22. FINAL VERIFICATION
==================================================

Setelah semua perbaikan selesai, verifikasi alur penuh:

REAL REQUEST
→ PROVIDER
→ MODEL
→ TOKEN USAGE
→ USAGE RECORD
→ PRICING LOOKUP
→ INPUT COST
→ OUTPUT COST
→ TOTAL COST
→ AGGREGATION
→ ADMIN API
→ DASHBOARD

Pastikan semua menggunakan source of truth yang sama.

==================================================
23. HASIL AKHIR
==================================================

Laporkan:

- root cause yang ditemukan
- file yang diperbaiki
- perubahan yang dilakukan
- token calculation
- pricing lookup
- input cost
- output cost
- total cost
- historical records yang berhasil diperbaiki
- record yang tetap N/A dan alasannya
- streaming handling
- dashboard calculation
- provider aggregation
- model aggregation
- test baru/perbaikan
- npm run lint
- npm run build
- npm test
- jumlah pass/fail/skip
- regression yang diverifikasi
- masalah yang masih tersisa

PENTING:

Jangan mengarang pricing.
Jangan mengarang token.
Jangan menganggap unknown pricing sebagai $0.
Jangan menghitung cost berbeda antara backend dan dashboard.
Jangan hanya audit — SETIAP BUG YANG DITEMUKAN DAN AMAN DIPERBAIKI HARUS LANGSUNG DIPERBAIKI.
Jangan membocorkan credential.
Jangan menjalankan test Gorouter.app.
```

# Prompt: Provider API Key — Full Integration Audit
```

Lakukan FULL AUDIT dan VERIFIKASI fitur Provider API Key pada project `nvidia-api` setelah implementasi ApiKeyStore, KeyManager, dan UI Provider API Key selesai.

TUJUAN:
Pastikan seluruh jalur API Key benar-benar terhubung dari UI → storage → provider → request → rotation → usage/logs, tanpa membocorkan secret dan tanpa merusak provider existing.

JANGAN melakukan refactor besar.
JANGAN membuat provider/key dummy.
JANGAN mengubah behavior provider yang sudah bekerja kecuali diperlukan untuk memperbaiki integrasi API key.

1. AUDIT STORAGE API KEY

Audit:
- `src/lib/api-key-store.ts`
- `src/lib/key-manager.ts`
- seluruh pemanggil ApiKeyStore/KeyManager.

Pastikan setiap provider dapat memiliki multiple API key.

Minimal data internal:
- provider
- key identifier/id
- masked key
- status active/disabled jika memang didukung
- createdAt
- updatedAt jika tersedia

API key RAW hanya boleh berada di storage internal yang memang diperlukan.

Jangan expose raw key melalui:
- API response
- dashboard
- logs
- error message
- backup
- browser/client JavaScript.

2. TAMBAH API KEY MELALUI UI

Audit flow:

Admin UI
→ pilih provider
→ masukkan API key
→ submit
→ backend validation
→ ApiKeyStore
→ KeyManager

Pastikan:
- key tersimpan benar
- provider benar
- key tidak hilang setelah restart
- duplicate key ditangani secara idempotent
- UI tidak menampilkan raw key setelah disimpan
- input API key menggunakan password/secret-style field jika sesuai UI existing.

Setelah berhasil:
UI cukup menampilkan:
- masked key
- status
- jumlah key
- created date jika tersedia.

3. DELETE API KEY

Pastikan admin dapat menghapus API key tertentu.

Flow:
UI
→ DELETE endpoint
→ validasi provider/key ID
→ hapus dari ApiKeyStore
→ KeyManager memperbarui pool.

Pastikan:
- key benar-benar tidak digunakan lagi setelah dihapus
- key lain pada provider yang sama tidak ikut terhapus
- tidak menghapus provider
- tidak menghapus env fallback key jika desain existing memang memisahkannya
- delete key terakhir ditangani dengan aman.

Jika key sedang aktif digunakan oleh request:
- jangan menyebabkan request yang sedang berjalan crash.
- perubahan berlaku untuk request berikutnya.

4. JUMLAH API KEY

Pastikan UI menampilkan jumlah API key per provider.

Contoh:

NVIDIA
API Keys: 3

Jumlah harus berasal dari storage aktual, bukan hardcode.

Pastikan count konsisten antara:
- UI
- endpoint admin
- ApiKeyStore
- KeyManager.

5. MULTIPLE KEY ROTATION

Audit KeyManager existing.

Pastikan jika provider memiliki:

KEY A
KEY B
KEY C

request berjalan menggunakan pool key tersebut sesuai mekanisme round-robin yang sudah ada.

Pastikan:
- tidak selalu memakai key pertama
- key yang tersedia dapat digunakan bergantian
- key disabled/deleted tidak dipilih
- pool otomatis berubah setelah add/delete.

Jangan membuat rotation system kedua.

Gunakan `KeyManager` existing.

6. PROVIDER FALLBACK / ENV KEY

Audit hubungan:
- API key dari UI/store
- API key dari environment/configuration.

Pastikan behavior existing tidak rusak.

Jika provider memiliki key dari UI:
- gunakan key sesuai KeyManager.

Jika tidak ada key UI:
- gunakan env key hanya jika memang behavior existing mengizinkannya.

Jangan menghapus env key.

Jangan memasukkan env key ke UI sebagai raw value.

Jangan membuat duplicate key hanya karena env key dan UI key memiliki nilai sama.

7. REQUEST NYATA

Gunakan provider/model yang memang sudah tersedia dan valid pada environment.

Lakukan request nyata melalui:

`POST /v1/chat/completions`

Pastikan request:
- memilih provider yang benar
- memilih model yang benar
- mengambil API key dari KeyManager
- request berhasil jika credential valid.

Jangan menggunakan provider dummy.
Jangan menggunakan Gorouter.app sebagai fallback.

8. API KEY FAILURE / ROTATION

Jika memungkinkan secara aman, verifikasi behavior ketika salah satu key gagal.

Contoh:

KEY A → upstream authorization error
KEY B → valid

Pastikan KeyManager dapat berpindah ke key berikutnya jika mekanisme retry/rotation existing memang mendukungnya.

Jangan membuat retry baru jika arsitektur existing tidak mendukung retry.

Yang penting:
- jangan retry tanpa batas
- jangan menyebabkan request loop
- jangan menyembunyikan error upstream sebenarnya.

9. USAGE TRACKING

Setelah request menggunakan API key provider, audit Usage Log.

Pastikan Usage Log menyimpan:
- provider
- model
- status
- HTTP status
- input tokens
- output tokens
- total tokens
- latency
- timestamp
- client/API identifier yang sudah masked.

JANGAN menyimpan raw provider API key.

Jika sistem memang memiliki key ID:
- boleh mencatat key ID/identifier yang aman
- jangan mencatat raw key.

10. ADMIN UI

Audit halaman provider.

Pastikan setiap provider memiliki:

- Provider name
- Provider status
- API Keys count
- daftar API key masked
- Add API Key
- Delete API Key
- status key jika tersedia.

UI harus tetap bersih dan konsisten dengan dashboard existing.

Pastikan:
- tombol Add bekerja
- tombol Delete bekerja
- confirmation sebelum delete jika UI existing menggunakan modal confirmation
- loading state
- success/error feedback
- empty state ketika tidak ada key.

11. API ENDPOINT SECURITY

Audit endpoint API key management.

Pastikan endpoint:
- hanya dapat digunakan oleh admin
- tidak dapat diakses public `/v1/*`
- validasi provider
- validasi key ID
- tidak mengembalikan raw API key.

Periksa:
- GET/list keys
- POST/add key
- DELETE key
- count keys

Gunakan endpoint existing jika sudah tersedia.
Jangan membuat endpoint duplikatif.

12. RESTART PERSISTENCE

Test:

add key
→ restart server
→ cek UI
→ cek count
→ request

Pastikan key tetap tersedia setelah restart.

Jangan kehilangan data karena hanya tersimpan di memory.

13. DELETE PERSISTENCE

Test:

add KEY A
add KEY B
delete KEY A
restart server

Pastikan:
- KEY A tetap terhapus
- KEY B tetap ada
- count benar
- KeyManager hanya menggunakan KEY B.

14. SECURITY AUDIT

Lakukan audit source code dan runtime untuk mencari kemungkinan secret leak.

Cari raw API key pada:
- console.log
- logger
- error
- response JSON
- admin endpoint
- frontend state
- browser response
- backup
- test output.

Pastikan tidak ada raw key yang bocor.

Gunakan masking seperti:
`sk-****1111`

atau mekanisme masking existing.

Jangan menampilkan secret dalam laporan akhir.

15. BACKUP COMPATIBILITY

Pastikan API key provider TIDAK masuk backup plaintext.

Backup boleh menyimpan:
- provider state
- key metadata/count jika memang diperlukan.

Jangan menyimpan:
- raw API key
- Authorization header
- provider secret.

Restore tidak boleh membuat credential palsu.

16. TESTING

Tambahkan atau audit test untuk:

- add API key
- duplicate API key
- list API keys
- count API keys
- delete API key
- delete unknown key
- multiple keys
- round-robin KeyManager
- deleted key tidak dipilih
- restart persistence
- provider association
- admin authorization
- raw key masking
- raw key tidak masuk response
- raw key tidak masuk logs
- env key compatibility
- Usage tetap tercatat
- request menggunakan key dari KeyManager.

Jalankan:

npm run lint
npm run build
npm test

PENTING:
Jangan menjalankan test/integration test Gorouter.app.

Jika test suite otomatis memuat test Gorouter:
- skip/exclude test tersebut
- jangan mengubah test Gorouter agar lulus
- laporkan jumlah test yang di-skip.

NVIDIA dan TokenHarbor boleh dites sesuai kebutuhan existing project.

17. REGRESSION

Pastikan tidak merusak:

- Provider Management
- Enable/Disable Provider
- Model Registry
- `/v1/models`
- `/v1/chat/completions`
- streaming
- Usage Tracking
- Usage Dashboard
- Logs
- Backup
- Restore
- existing provider authentication
- env API key fallback.

18. HASIL AKHIR

Berikan laporan lengkap:

API KEY STORAGE
- file yang digunakan
- struktur data
- persistence

UI
- Add API Key
- Delete API Key
- API Key count
- masked display

KEY MANAGER
- rotation
- multiple keys
- deleted/disabled key handling

REQUEST
- provider yang digunakan
- model yang digunakan
- API key source: UI/store atau env
- HTTP status
- success/error

USAGE
- provider
- model
- input tokens
- output tokens
- total tokens
- latency
- API key identifier jika ada

SECURITY
- raw key leak audit
- logs
- API response
- backup

TEST
- pass
- fail
- skipped
- lint
- build

JANGAN:
- membocorkan API key
- mencetak credential
- membuat provider/model dummy
- menggunakan Gorouter sebagai fallback
- mengubah test hanya agar lulus
- melakukan refactor besar.

```

# Prompt 14 — Provider API Key Management
```

Implementasikan fitur **API Key Management per Provider** pada project `nvidia-api`.

TUJUAN:

Admin harus dapat mengelola API key provider langsung dari UI Admin:

Provider
→ API Keys
→ Add API Key
→ Delete API Key
→ Enable/Disable jika architecture mendukung
→ Jumlah API Key
→ Status/health jika sudah tersedia

Jangan membuat sistem provider baru.
Gunakan Provider Management dan struktur provider existing.

1. AUDIT ARSITEKTUR TERLEBIH DAHULU

Sebelum coding, audit:

- provider registry
- provider configuration
- environment/config
- authentication provider
- request routing
- provider.ts
- admin API
- admin dashboard
- persistence/storage
- backup/restore

Cari bagaimana credential provider saat ini disimpan dan digunakan.

JANGAN langsung membuat schema baru sebelum memahami storage existing.

2. DATA MODEL API KEY

Buat struktur API key yang aman.

Minimal setiap API key memiliki:

- id
- providerId
- maskedKey
- createdAt
- updatedAt
- status jika diperlukan
- metadata non-secret jika memang diperlukan

RAW API KEY hanya boleh disimpan pada storage yang memang diperlukan untuk runtime authentication.

Jangan pernah menyimpan atau menampilkan raw key pada response Admin API.

Contoh UI:

Provider: NVIDIA
API Keys: 3

- sk-••••••••1234    Active
- sk-••••••••5678    Active
- sk-••••••••9012    Disabled

Jangan tampilkan full API key.

3. ADD API KEY MELALUI UI

Tambahkan tombol:

`+ Add API Key`

Flow:

Provider
→ Add API Key
→ input API Key
→ optional label jika architecture membutuhkan
→ Save

Setelah berhasil:

- key tersimpan persistent
- UI menampilkan masked key
- jumlah API key bertambah
- raw key tidak ditampilkan kembali
- tidak masuk browser console
- tidak masuk API response
- tidak masuk logs.

Validasi:

- key tidak boleh kosong
- trim whitespace
- provider harus valid
- duplicate key untuk provider yang sama harus ditangani dengan aman
- jangan menyimpan malformed/empty credential.

4. DELETE API KEY

Tambahkan tombol:

`Delete`

Sebelum delete gunakan confirmation modal.

Contoh:

Delete API Key?

Provider: NVIDIA
Key: sk-••••••••1234

[Cancel] [Delete]

Setelah confirm:

- hapus key dari persistent storage
- key tidak dapat digunakan untuk request berikutnya
- jumlah API key langsung diperbarui
- UI refresh tanpa reload penuh jika memungkinkan.

Jangan menghapus provider.

Jangan menghapus model.

Jangan menghapus Usage Logs.

5. JUMLAH API KEY

Pada setiap provider tampilkan:

`API Keys: N`

Contoh:

NVIDIA
Active
Models: 38
API Keys: 3

Jumlah harus berasal dari backend/storage, bukan hardcoded.

Pastikan count konsisten antara:

storage
→ admin API
→ dashboard.

Jika provider tidak memiliki key:

`API Keys: 0`

Jangan menampilkan raw credential.

6. API KEY LIST ENDPOINT

Jika architecture menggunakan Admin API, tambahkan endpoint yang konsisten dengan existing.

Contoh konsep:

GET `/admin/providers/:providerId/api-keys`

Response hanya boleh berisi data aman:

- id
- providerId
- maskedKey
- status
- createdAt
- updatedAt

JANGAN mengembalikan:

- raw API key
- Authorization header
- secret
- provider credential lainnya.

7. ADD ENDPOINT

Tambahkan endpoint Admin untuk membuat API key jika sesuai architecture.

Input:

- providerId
- apiKey
- label jika diperlukan

Response:

- success
- id
- maskedKey
- status
- timestamps

Raw API key TIDAK BOLEH dikembalikan.

8. DELETE ENDPOINT

Tambahkan endpoint Admin untuk menghapus key.

Validasi:

- provider exists
- key exists
- key memang milik provider tersebut

Jika tidak ditemukan:

HTTP 404 dengan error yang jelas.

Jangan menghapus key provider lain karena ID collision/manipulation.

9. RUNTIME PROVIDER AUTHENTICATION

Ini bagian paling penting.

Audit bagaimana request saat ini mendapatkan credential provider.

API key yang ditambahkan melalui UI harus benar-benar dapat digunakan oleh runtime provider jika provider tersebut memang mendukung multiple API keys.

Jangan membuat storage key yang hanya tampil di UI tetapi tidak digunakan runtime.

Pastikan:

Add Key
→ persistent storage
→ provider runtime
→ request dapat menggunakan key tersebut.

Delete Key
→ persistent storage
→ runtime tidak lagi menggunakan key tersebut.

10. MULTIPLE API KEYS

Jika architecture provider mendukung multiple credential:

Provider NVIDIA:

Key A
Key B
Key C

Request baru harus dapat memilih key berdasarkan mekanisme routing existing.

Jangan membuat round-robin baru jika project sudah memiliki mekanisme credential rotation.

Jika belum ada rotation:

- jangan mengarang behavior
- gunakan satu key sesuai architecture existing
- dokumentasikan bahwa multiple-key storage sudah tersedia tetapi rotation belum diimplementasikan.

JANGAN mengubah routing besar hanya untuk fitur ini.

11. ENABLE/DISABLE API KEY

Jika mudah diintegrasikan dengan architecture existing, tambahkan:

Enable
Disable

per API key.

Disabled key:

- tidak boleh digunakan untuk request baru
- tetap tersimpan
- dapat di-enable kembali.

Jika project belum memiliki konsep status credential, jangan melakukan refactor besar hanya untuk menambahkannya.

12. API KEY COUNT DI PROVIDER UI

Update Provider Management.

Setiap provider harus menampilkan:

- Provider
- Status
- Models
- API Keys count

Contoh:

NVIDIA
Active

Models: 38
API Keys: 3

[Manage API Keys]
[Disable]

Klik:

`Manage API Keys`

membuka modal/page:

NVIDIA API Keys

3 API Keys

[+ Add API Key]

Key 1  sk-••••1234  Active  [Delete]
Key 2  sk-••••5678  Active  [Delete]
Key 3  sk-••••9012  Disabled [Enable] [Delete]

13. SECURITY

WAJIB audit seluruh jalur secret.

Pastikan raw API key TIDAK muncul di:

- dashboard HTML
- API response
- browser localStorage
- browser sessionStorage
- console.log
- server logs
- Usage Logs
- error stack
- backup
- Git
- GitHub/GitLab
- debug output.

Authorization header juga tidak boleh disimpan pada Usage Logs.

Jika backup system sudah ada:

API key provider HARUS tetap tidak masuk backup plaintext.

14. ENV COMPATIBILITY

Jangan langsung menghapus API key provider yang sekarang berasal dari `.env`.

Audit compatibility.

Existing:

ENV API KEY
+
UI-managed API KEY

harus tetap dapat bekerja jika architecture memungkinkan.

Jangan memindahkan credential production secara otomatis tanpa migration yang aman.

Jika diperlukan migration:

- jangan hapus `.env`
- jangan overwrite credential
- jelaskan migration yang diperlukan.

15. PERSISTENCE

API key yang ditambahkan melalui UI harus tetap ada setelah:

- server restart
- PM2 restart
- deployment restart

Gunakan storage existing.

Jangan membuat database baru.

Jangan membuat JSON storage baru jika project sudah memiliki persistent storage yang sesuai.

16. USAGE INTEGRATION

Usage Logs harus tetap aman.

Jangan mencatat raw API key.

Jika usage memang perlu mengidentifikasi credential:

gunakan:

- key ID
- masked key
- credential identifier

bukan raw secret.

Contoh:

provider = nvidia
apiKeyId = key_123
apiKeyMasked = sk-••••1234

17. DELETE SAFETY

Sebelum delete:

- pastikan key benar-benar milik provider
- confirmation diperlukan
- jangan delete provider
- jangan delete model
- jangan delete Usage Records.

Jika key sedang digunakan oleh request aktif:

jangan memutus request yang sedang berjalan jika architecture dapat menghindarinya.

Delete berlaku untuk request baru.

18. PROVIDER DENGAN 0 API KEY

Provider tetap boleh tampil di Admin.

Contoh:

NVIDIA
Active
API Keys: 0

Tetapi request harus mengikuti behavior existing ketika tidak ada credential:

- error yang jelas
- jangan crash server
- jangan menggunakan credential provider lain secara diam-diam.

19. TESTING

Tambahkan test untuk:

API KEY CRUD:

- add key
- add empty key
- add invalid provider
- duplicate key
- list keys
- delete key
- delete nonexistent key
- delete key dari provider lain

COUNT:

- zero keys
- one key
- multiple keys
- count setelah add
- count setelah delete

SECURITY:

- raw key tidak muncul response
- raw key tidak muncul logs
- raw key tidak muncul Usage Logs
- raw key tidak masuk backup
- masked key benar

PERSISTENCE:

- key tetap ada setelah restart/reload storage

RUNTIME:

- added key dapat digunakan oleh provider jika architecture mendukung
- deleted key tidak digunakan request baru
- disabled key tidak digunakan jika status key diterapkan

PROVIDER REGRESSION:

- provider enable/disable tetap bekerja
- model registry tetap bekerja
- `/v1/models` tetap bekerja
- normal request tetap bekerja
- streaming tetap bekerja
- usage tracking tetap bekerja
- usage cost tetap bekerja
- dashboard tetap bekerja.

20. UI TEST

Pastikan UI:

- Add API Key modal bekerja
- input password/secret type digunakan untuk API key
- key tidak terlihat setelah save
- masked key ditampilkan
- Delete confirmation bekerja
- count API key langsung berubah
- error backend ditampilkan dengan jelas
- loading state tersedia
- duplicate/error state ditangani.

21. NO SECRET LEAK AUDIT

Setelah implementasi lakukan pencarian source/log untuk memastikan tidak ada pola seperti:

console.log(apiKey)
console.log(config)
console.log(headers)
JSON.stringify(providerConfig)

yang dapat membocorkan credential.

Periksa juga:

- error handler
- request logging
- admin API
- backup
- Usage Logs.

22. TEST COMMAND

Jalankan:

npm run lint

npm run build

npm test

Jangan mengubah test hanya agar menjadi hijau.

Jangan menjalankan atau memicu test/integration test Gorouter.app.

Jika ada test Gorouter yang otomatis ikut dijalankan:
- skip/exclude sesuai mekanisme existing
- laporkan jumlah test yang di-skip.

23. REGRESSION

Pastikan tidak merusak:

- Provider Management
- Enable/Disable Provider
- Model Registry
- `/v1/models`
- API request
- streaming
- Usage Tracking
- Usage Dashboard
- Usage Cost
- Logs
- Backup/Restore
- NVIDIA
- TokenHarbor.ai

Jangan melakukan refactor besar.

24. HASIL AKHIR

Setelah selesai laporkan:

1. Storage API key yang digunakan.
2. Schema/data model API key.
3. Endpoint Add.
4. Endpoint List.
5. Endpoint Delete.
6. Enable/Disable key jika dibuat.
7. API key count.
8. UI Add API Key.
9. UI Delete API Key.
10. Masking.
11. Runtime authentication integration.
12. Persistence setelah restart.
13. Backup security.
14. Usage security.
15. File yang diubah.
16. Test yang ditambahkan.
17. npm run lint.
18. npm run build.
19. npm test.
20. Jumlah pass/fail/skip.
21. Masalah yang masih tersisa.

ACCEPTANCE CRITERIA:

Admin UI
→ Add API Key
→ Key tersimpan aman
→ Key muncul sebagai masked
→ API Key count bertambah
→ Runtime provider dapat menggunakan key sesuai architecture
→ Delete
→ Key hilang dari runtime request baru
→ API Key count berkurang

DAN:

Raw API key TIDAK BOLEH pernah muncul pada UI setelah save, API response, logs, Usage, backup, atau output debugging.

Fokus pada **API Key Management per Provider**.
Jangan mengubah arsitektur provider secara besar.
Jangan membuat provider/model dummy.
Jangan mengarang credential.
Jangan membocorkan secret.

```


# Prompt 13 — Usage End-to-End Cost Verification
```

Lakukan FINAL END-TO-END AUDIT untuk sistem Usage dan Cost pada project nvidia-api.

TUJUAN UTAMA:

Pastikan seluruh jalur:

API Request
→ Provider
→ Exact Model
→ Upstream Usage
→ Input Tokens
→ Output Tokens
→ Total Tokens
→ Model Pricing
→ Cost USD
→ Usage Store
→ Aggregation
→ Admin API
→ Dashboard

menggunakan satu sumber data dan menghasilkan nilai yang konsisten.

FOKUS:

Jangan mengubah atau mengaudit model discovery/provider discovery kecuali diperlukan langsung untuk memperbaiki Usage/Cost.

Jangan membuat model/provider dummy.
Jangan mengarang token.
Jangan menganggap model unknown sebagai free.
Jangan menganggap cost null sebagai $0.
Jangan menggunakan Gorouter.app.

1. AUDIT SELURUH USAGE PATH

Cari semua lokasi code yang:

- menerima usage dari upstream
- membaca prompt/input tokens
- membaca completion/output tokens
- membaca total tokens
- membaca cached tokens
- mencari pricing model
- menghitung cost
- membuat Usage Record
- menyimpan costUsd
- melakukan aggregation
- menyediakan data ke admin dashboard

Minimal audit:

- src/lib/pricing.ts
- src/lib/usage-store.ts
- src/services/stream-usage.ts
- src/services/provider.ts
- src/routes/admin.ts
- src/admin/dashboard.ts
- src/admin/index.html
- src/lib/model-registry.ts
- seluruh request handler
- seluruh endpoint Usage
- seluruh fungsi aggregation

Temukan apakah ada lebih dari satu implementation untuk menghitung cost.

Jika ada formula cost berbeda, satukan ke implementation yang benar tanpa refactor besar.

2. TOKEN SOURCE OF TRUTH

Token harus berasal dari usage response upstream jika tersedia.

Gunakan:

- prompt_tokens / input_tokens
- completion_tokens / output_tokens
- total_tokens
- cached input tokens jika tersedia

Jika upstream memberikan total_tokens:

validasi:

total_tokens = input_tokens + output_tokens

Jika upstream tidak memberikan usage:

- simpan null sesuai schema existing
- jangan estimasi token
- jangan menghitung token dari panjang prompt/completion
- jangan membuat angka dummy

3. MODEL PRICING

Audit mapping:

exact provider + exact model ID
→ pricing

Pastikan pricing tidak salah karena:

- alias model
- nama display
- model family
- fallback model
- prefix yang salah

Jika pricing model tidak tersedia:

costUsd harus tetap null.

JANGAN:

unknown pricing → $0
unknown model → $0
free name → otomatis $0

Model hanya boleh dianggap gratis jika pricing registry secara eksplisit mempunyai harga $0.

4. COST CALCULATION

Jika pricing menggunakan USD per 1M tokens:

inputCost =
inputTokens / 1,000,000 × inputPricePerM

outputCost =
outputTokens / 1,000,000 × outputPricePerM

cachedInputCost =
cachedInputTokens / 1,000,000 × cachedInputPricePerM

totalCost =
inputCost + outputCost + cachedInputCost + komponen lain yang memang tersedia.

Pastikan cached input tidak dihitung dua kali sebagai regular input.

5. CACHED INPUT PRICING

Audit khusus field:

cachedInputPricePerM

Pastikan:

- cached token tersedia + cached pricing tersedia → dihitung
- cached token tidak tersedia → jangan membuat cost
- cached token tidak dihitung sebagai regular input sekaligus cached input
- tidak terjadi double counting

Jika schema existing belum mendukung cached usage dengan benar, lakukan perbaikan minimal.

6. FLOATING POINT PRECISION

Perbaiki masalah seperti:

0.5636899999999999

Cost harus konsisten pada:

calculation
→ storage
→ API
→ aggregation
→ dashboard

Jangan hanya memperbaiki tampilan frontend dengan toFixed().

Source of truth calculation/storage harus benar.

Jangan mengubah numeric cost menjadi string kecuali memang diperlukan oleh architecture existing.

Audit juga record lama yang sudah memiliki floating-point artifact.

Jangan mengubah historical record secara sembarangan.

7. costUsd NULL

Audit seluruh record yang memiliki:

costUsd: null

Khusus temuan sebelumnya:

sekitar 151 production records memiliki costUsd null karena model belum tersedia di pricing registry.

Pastikan:

- null tetap null jika pricing tidak dapat diketahui
- tidak diubah menjadi $0
- dashboard tidak memasukkan null sebagai biaya $0

Dashboard harus dapat menunjukkan:

Known Cost
Unknown Cost Records

Contoh:

Request A = $1
Request B = null
Request C = $2

Maka:

Known Cost = $3
Unknown Cost Records = 1

Bukan:

Total Cost = $3 dengan asumsi request B = $0.

8. HISTORICAL BACKFILL

Jangan melakukan mass backfill historical record hanya agar dashboard terlihat lengkap.

Backfill hanya boleh dilakukan jika:

exact model
+
provider
+
pricing yang valid
+
aturan historical pricing yang jelas

memungkinkan cost dihitung secara deterministik.

Jika tidak yakin:

pertahankan null.

Laporkan jumlah record yang tetap null dan alasan sebenarnya.

9. SUCCESS REQUEST

Audit request sukses.

Contoh:

input = 1,000
output = 500
total = 1,500

Jika:

input price = $1 / 1M
output price = $2 / 1M

Expected:

inputCost = $0.001
outputCost = $0.001
totalCost = $0.002

Tambahkan automated test untuk memastikan hasil tersebut.

10. ERROR REQUEST

Audit:

- upstream error
- validation error
- blocked request
- invalid model

Jika tidak ada upstream inference:

jangan membuat token/cost palsu.

Jika upstream memberikan usage sebelum error dan architecture memang menyimpannya:

gunakan usage tersebut.

Jika usage tidak tersedia:

token = null
cost = null

11. STREAMING

Audit:

src/services/stream-usage.ts

Pastikan streaming menggunakan usage asli dari final response/chunk jika tersedia.

Jika:

usage = null

maka:

tokens = null
cost = null

Jangan estimasi token.

Logging usage tidak boleh:

- memutus stream
- mengubah response
- menyebabkan request gagal

Pastikan streaming dan non-streaming menggunakan cost calculation yang sama.

12. USAGE STORE

Audit Usage Store.

Pastikan record menyimpan secara konsisten:

- timestamp
- provider
- exact model
- status
- HTTP status
- input tokens
- output tokens
- total tokens
- cached tokens jika tersedia
- costUsd
- latency
- error information
- request/client identifier yang sudah aman

Cost harus dihitung sekali dan tidak dihitung ulang dengan formula berbeda ketika dibaca.

13. AGGREGATION

Audit semua aggregation.

Pastikan:

provider total
=
SUM cost dari record yang memiliki known cost

model total
=
SUM cost dari record yang memiliki known cost

overall total
=
SUM known cost

Jangan diam-diam mengubah null menjadi $0 tanpa menyediakan informasi unknown cost.

Audit:

- total requests
- successful
- failed
- blocked
- input tokens
- output tokens
- total tokens
- cost
- unknown cost count

14. PROVIDER BREAKDOWN

Audit endpoint:

/admin/usage/providers

Pastikan setiap provider menampilkan:

- provider
- requests
- success
- error
- blocked
- input tokens
- output tokens
- total tokens
- known cost
- unknown cost count

Cost provider tidak boleh tercampur dengan provider lain.

15. MODEL BREAKDOWN

Audit:

/admin/usage/models

Pastikan setiap model menampilkan:

- exact model
- provider
- requests
- input tokens
- output tokens
- total tokens
- known cost
- unknown cost count

Pricing berdasarkan exact model ID.

16. USAGE RECORD DETAIL

Audit:

/admin/usage/records

Pastikan detail record menunjukkan:

- timestamp
- provider
- model
- input tokens
- output tokens
- total tokens
- cached tokens jika tersedia
- costUsd
- status
- HTTP status
- latency

Jangan expose API key atau Authorization header.

17. ADMIN DASHBOARD

Dashboard harus mengambil cost dari backend.

JANGAN menghitung ulang cost di frontend.

Dashboard minimal menampilkan:

Usage Summary:

- Total Requests
- Successful
- Failed
- Blocked
- Input Tokens
- Output Tokens
- Total Tokens
- Known Cost
- Unknown Cost Records

Provider:

- Provider
- Requests
- Tokens
- Cost
- Unknown Cost

Model:

- Model
- Provider
- Requests
- Tokens
- Cost
- Unknown Cost

18. DASHBOARD CONSISTENCY

Ambil beberapa record nyata dari Usage Store.

Bandingkan:

Database
→ Usage API
→ Provider aggregation
→ Model aggregation
→ Dashboard

Nilainya harus konsisten.

Contoh:

Database:
costUsd = 0.002

Maka:

Usage API = 0.002
Provider = 0.002
Model = 0.002
Dashboard = $0.002

Tidak boleh ada perbedaan.

19. TESTING

Tambahkan test untuk:

TOKEN:

- input token
- output token
- total token
- null usage
- cached token

PRICING:

- known model
- unknown model
- known free model
- unknown pricing
- cached pricing

COST:

- exact calculation
- zero cost
- fractional cost
- precision
- large token count
- null pricing
- null usage

REQUEST:

- success
- upstream error
- blocked
- invalid model

STREAM:

- usage tersedia
- usage null

STORAGE:

- cost tersimpan benar
- precision benar
- null tetap null

AGGREGATION:

- total cost
- provider cost
- model cost
- unknown cost
- null tidak menjadi $0

DASHBOARD:

- summary
- provider
- model
- known cost
- unknown cost
- total tokens

20. REGRESSION

Pastikan tidak merusak:

- Provider Management
- Enable/Disable Provider
- Model Registry
- /v1/models
- normal API request
- streaming
- Usage Tracking
- Usage Logs
- Usage Dashboard
- Backup/Restore

Jangan melakukan refactor besar.

21. TEST COMMAND

Jalankan:

npm run lint

npm run build

npm test

JANGAN menjalankan atau memicu test Gorouter.app.

Jangan mengubah test hanya untuk membuat hasil hijau.

Jika terdapat failure yang berasal dari test/environment yang sudah ada, laporkan sebagai pre-existing jika memang terbukti demikian.

22. FINAL AUDIT REPORT

Setelah selesai, WAJIB laporkan:

1. Semua Usage path yang diaudit.
2. Source of truth token.
3. Source of truth pricing.
4. Source of truth cost.
5. Formula cost.
6. Cached input pricing.
7. Floating-point precision.
8. Jumlah record costUsd null.
9. Apakah historical record diubah.
10. Known Cost.
11. Unknown Cost.
12. Provider aggregation.
13. Model aggregation.
14. Dashboard aggregation.
15. Streaming cost handling.
16. Error/blocked cost handling.
17. File yang diubah.
18. Test yang ditambahkan.
19. npm run lint result.
20. npm run build result.
21. npm test result.
22. Jumlah test pass/fail/skip.
23. Masalah yang masih tersisa.

ACCEPTANCE CRITERIA:

API Request
→ Real Upstream Usage
→ Exact Model
→ Correct Pricing
→ Correct Cost
→ Usage Store
→ Aggregation
→ Admin API
→ Dashboard

HARUS menghasilkan nilai yang sama.

Tidak boleh ada:

- fake token
- fake pricing
- unknown model dianggap free
- null cost dianggap $0
- cached token double counted
- cost dihitung dua kali
- frontend menghitung cost berbeda
- floating-point artifact yang tidak perlu
- historical pricing yang ditebak

Fokus hanya pada Usage → Token → Pricing → Cost → Dashboard.

```
# Prompt: Full Usage Flow Audit — Token → Cost → Dashboard
```
Lakukan FULL AUDIT dan perbaikan menyeluruh terhadap seluruh jalur Usage pada project `nvidia-api`.

TUJUAN UTAMA:

Pastikan setiap request yang berhasil diproses mempunyai alur Usage yang benar:

REQUEST
→ PROVIDER
→ MODEL
→ UPSTREAM RESPONSE
→ INPUT TOKENS
→ OUTPUT TOKENS
→ TOTAL TOKENS
→ MODEL/PROVIDER PRICING
→ TOTAL COST/HARGA
→ USAGE STORAGE
→ AGGREGATION
→ ADMIN USAGE DASHBOARD

Jangan hanya memperbaiki UI dashboard.
Audit dari sumber data paling awal sampai data yang akhirnya ditampilkan di dashboard.

==================================================
ATURAN PENTING
==================================================

1. Fokus utama HANYA pada Usage, Token Accounting, Pricing/Cost, dan Dashboard Usage.

2. JANGAN mengubah Model Registry/provider discovery hanya karena test models bermasalah.

3. Model yang sudah tersedia harus dipakai sebagai sumber metadata model.
   Jangan membuat model dummy.

4. Jangan membuat mock usage untuk membuat test lulus.

5. Jangan mengarang jumlah token.

6. Token harus berasal dari usage response/provider jika tersedia.

7. Jangan menghitung token berdasarkan panjang teks kecuali project memang sudah mempunyai mekanisme resmi yang sengaja digunakan untuk itu.

8. Jangan menggunakan floating point untuk perhitungan uang jika dapat dihindari.
   Gunakan integer minor units atau Decimal/precision-safe calculation sesuai stack existing.

9. Jangan mengubah behavior API request yang sudah berjalan kecuali memang diperlukan untuk memperbaiki Usage.

10. Jangan membocorkan API key, Authorization header, credential provider, atau secret.

==================================================
PHASE 1 — AUDIT END-TO-END USAGE FLOW
==================================================

Audit seluruh source code dan cari semua jalur yang berhubungan dengan:

- usage
- usage tracking
- usage store
- usage repository
- usage service
- recordUsage
- token accounting
- prompt tokens
- input tokens
- completion tokens
- output tokens
- total tokens
- cached tokens jika ada
- pricing
- cost
- price
- amount
- dashboard usage
- admin usage
- usage providers
- usage models
- usage records
- logs
- request tracking
- streaming usage

Identifikasi dengan jelas:

A. Di mana request masuk.
B. Di mana provider dipilih.
C. Di mana model ditentukan.
D. Di mana upstream response diterima.
E. Di mana usage token dibaca.
F. Di mana usage dinormalisasi.
G. Di mana usage disimpan.
H. Di mana harga model/provider ditentukan.
I. Di mana cost dihitung.
J. Di mana aggregation dilakukan.
K. Di mana dashboard mengambil data.

Buat diagram/alur singkat berdasarkan source code aktual.

Jangan berasumsi.

==================================================
PHASE 2 — AUDIT TOKEN SOURCE
==================================================

Pastikan setiap request yang menghasilkan usage memiliki:

- provider
- model
- inputTokens
- outputTokens
- totalTokens

Jika provider response menggunakan nama:

`prompt_tokens`
`completion_tokens`
`total_tokens`

normalisasi ke schema internal existing.

Jika provider menggunakan:

`input_tokens`
`output_tokens`

normalisasi dengan benar.

Jika provider mempunyai:

`cached_tokens`
`cache_read_input_tokens`
atau field sejenis,

audit apakah field tersebut sudah didukung.

Jangan menghilangkan data token tambahan yang memang tersedia.

==================================================
PHASE 3 — VALIDASI TOTAL TOKEN
==================================================

Untuk non-streaming request:

Jika input dan output tersedia:

totalTokens harus konsisten dengan:

inputTokens + outputTokens

Jika upstream memberikan total_tokens:

bandingkan:

upstream total_tokens
vs
inputTokens + outputTokens

Jika berbeda:

JANGAN diam-diam memperbaiki dengan angka buatan.

Cari penyebab sebenarnya, misalnya:

- cached tokens
- reasoning tokens
- provider-specific token accounting
- hidden/system tokens
- field mapping salah

Dokumentasikan behavior provider tersebut.

Jika upstream tidak memberikan total token:

gunakan perhitungan hanya jika memang secara semantik aman:

totalTokens = inputTokens + outputTokens

Jika input/output juga tidak tersedia:

totalTokens = null

Jangan membuat estimasi.

==================================================
PHASE 4 — STREAMING USAGE
==================================================

Audit streaming secara terpisah.

Periksa:

- initial chunks
- intermediate chunks
- final chunk
- usage field
- usage null
- stream completion
- stream error
- logging setelah stream selesai

Pastikan:

1. Streaming response tetap dikirim ke client.
2. Usage logging tidak memutus stream.
3. Jika final chunk memberikan usage → simpan usage.
4. Jika final chunk usage = null → jangan membuat token palsu.
5. Jika provider tidak menyediakan usage streaming → simpan null sesuai schema.
6. Jangan menghitung token dari text stream.
7. Jangan membuat request streaming menjadi gagal hanya karena usage tidak tersedia.

Pastikan streaming tidak menghasilkan duplicate usage records.

==================================================
PHASE 5 — AUDIT PRICING SYSTEM
==================================================

Cari sumber pricing yang saat ini digunakan project.

Periksa apakah pricing sudah tersedia berdasarkan:

provider + model

atau:

model

atau konfigurasi lainnya.

Jangan membuat pricing hardcoded di dashboard.

Pricing harus mempunyai satu source of truth.

Jika pricing belum ada, buat struktur pricing yang modular dan mudah diperluas.

Minimal support:

- input token price
- output token price

Jika provider/model mempunyai cached input pricing dan project memang menyimpan cached tokens, support:

- cached input token price

Pricing harus bisa berbeda untuk setiap model.

Contoh struktur konseptual:

provider
model
inputPricePer1M
outputPricePer1M
cachedInputPricePer1M
currency

Jangan menggunakan contoh harga tersebut sebagai harga nyata.
Gunakan pricing yang benar-benar tersedia di project/configuration.

==================================================
PHASE 6 — COST CALCULATION
==================================================

Implementasikan perhitungan cost berdasarkan token usage aktual.

Formula dasar:

inputCost =
(inputTokens / 1,000,000) × inputPricePer1M

outputCost =
(outputTokens / 1,000,000) × outputPricePer1M

totalCost =
inputCost + outputCost

Jika cached tokens didukung:

cachedCost =
(cachedTokens / 1,000,000) × cachedInputPricePer1M

dan gunakan aturan pricing provider/model yang benar.

PENTING:

Jangan menggunakan:

totalTokens × satu harga

jika provider mempunyai harga input/output berbeda.

Harga harus dihitung berdasarkan jenis token.

Simpan cost dengan precision yang aman.

Jangan melakukan:

Math.round()
atau floating-point calculation
yang menyebabkan kehilangan precision uang.

==================================================
PHASE 7 — CURRENCY
==================================================

Audit currency yang digunakan project.

Jika pricing menggunakan USD:

simpan cost dalam USD secara canonical.

Jangan melakukan konversi IDR hanya di backend secara hardcoded.

Jika dashboard ingin menampilkan IDR dan project memang memiliki exchange-rate mechanism:

pisahkan:

provider cost
→ canonical currency
→ display currency

Jangan mencampur token cost dengan kurs.

Jika belum ada exchange-rate system:

dashboard minimal menampilkan currency asli pricing.

==================================================
PHASE 8 — USAGE STORAGE
==================================================

Audit schema/database/storage Usage.

Setiap Usage Record harus dapat menyimpan minimal:

- id
- timestamp
- provider
- model
- status
- HTTP status
- inputTokens
- outputTokens
- totalTokens
- inputCost
- outputCost
- totalCost
- currency
- latency
- request ID jika tersedia
- client/API key identifier yang sudah masked jika tersedia
- error information jika gagal

Jika field cost belum ada:

tambahkan migration/schema update sesuai storage existing.

Jangan membuat database baru.

Jangan membuat storage Usage kedua.

Gunakan storage existing.

==================================================
PHASE 9 — SUCCESS / ERROR / BLOCKED BILLING
==================================================

Audit kapan cost boleh dihitung.

SUCCESS:

Jika provider benar-benar memproses request dan usage tersedia:

→ record usage
→ token
→ cost

UPSTREAM ERROR:

Jika upstream memproses request tetapi error:

- simpan status error
- gunakan usage hanya jika provider benar-benar memberikan usage
- jangan mengarang token
- cost hanya dihitung jika usage valid.

BLOCKED:

Jika provider/model/request diblokir SEBELUM upstream dipanggil:

- status = blocked
- upstream tidak menerima request
- token = null/0 sesuai schema existing
- cost = 0/null sesuai semantics existing

VALIDATION ERROR:

Jika request ditolak sebelum inference:

- jangan charge token
- jangan membuat success usage.

Pastikan dashboard tidak menghitung blocked/validation sebagai paid usage.

==================================================
PHASE 10 — USAGE AGGREGATION
==================================================

Audit seluruh aggregation.

Dashboard harus menghitung dari Usage Records yang benar.

Minimal:

TOTAL REQUESTS

SUCCESSFUL REQUESTS

FAILED REQUESTS

BLOCKED REQUESTS

TOTAL INPUT TOKENS

TOTAL OUTPUT TOKENS

TOTAL TOKENS

TOTAL INPUT COST

TOTAL OUTPUT COST

TOTAL COST

AVERAGE LATENCY

Pastikan:

totalTokens aggregation
=
sum(record.totalTokens)

dan:

totalCost aggregation
=
sum(record.totalCost)

Jangan menghitung total cost dari total token global jika pricing model berbeda-beda.

Contoh:

Model A:
input $1 / 1M
output $2 / 1M

Model B:
input $5 / 1M
output $10 / 1M

Cost harus dihitung per record/model terlebih dahulu,
baru dijumlahkan.

==================================================
PHASE 11 — PROVIDER BREAKDOWN
==================================================

Audit:

`/admin/usage/providers`

Pastikan setiap provider menampilkan:

- provider
- requests
- success
- errors
- blocked
- input tokens
- output tokens
- total tokens
- input cost
- output cost
- total cost
- currency
- average latency

Cost provider harus merupakan SUM cost record yang benar-benar menggunakan provider tersebut.

==================================================
PHASE 12 — MODEL BREAKDOWN
==================================================

Audit:

`/admin/usage/models`

Setiap model harus menampilkan:

- provider
- model
- request count
- success
- error
- blocked
- input tokens
- output tokens
- total tokens
- input cost
- output cost
- total cost
- currency
- average latency

PENTING:

Model yang berbeda dengan pricing berbeda tidak boleh digabung menjadi satu cost rate.

==================================================
PHASE 13 — USAGE RECORDS
==================================================

Audit:

`/admin/usage/records`

Setiap record harus menunjukkan:

Timestamp
Provider
Model
Status
HTTP Status
Input Tokens
Output Tokens
Total Tokens
Input Cost
Output Cost
Total Cost
Currency
Latency

Jika cost tidak dapat dihitung karena pricing/token tidak tersedia:

tampilkan:

Cost = null

atau behavior schema existing yang paling tepat.

Jangan tampilkan `0` jika sebenarnya data tidak diketahui.

Bedakan:

- cost benar-benar $0
- cost belum dapat dihitung

==================================================
PHASE 14 — LOG DETAIL
==================================================

Audit `/admin/logs`.

Log detail harus konsisten dengan Usage Record.

Pastikan:

provider
model
tokens
status
HTTP status
latency
cost

berasal dari record yang sama.

Jangan sampai:

Usage Dashboard menunjukkan 1000 tokens

sementara Log menunjukkan 0 tokens.

==================================================
PHASE 15 — DASHBOARD UI
==================================================

Setelah backend Usage benar, baru audit UI.

Dashboard Usage harus menampilkan minimal:

┌─────────────────────────┐
│ Total Requests          │
├─────────────────────────┤
│ Total Input Tokens      │
│ Total Output Tokens     │
│ Total Tokens            │
├─────────────────────────┤
│ Total Cost              │
│ Input Cost              │
│ Output Cost             │
└─────────────────────────┘

Tambahkan breakdown:

Provider

Model

dan Logs.

Format cost harus jelas, misalnya:

$0.123456

Jangan membulatkan terlalu agresif sehingga nilai cost kehilangan akurasi.

Jika cost sangat kecil:

tetap tampilkan precision yang berguna.

==================================================
PHASE 16 — FILTER DASHBOARD
==================================================

Pastikan filter yang existing tetap bekerja:

- provider
- model
- status
- date range
- request ID jika tersedia

Periksa bahwa ketika filter diterapkan:

token aggregation berubah sesuai filter.

cost aggregation juga berubah sesuai filter.

Jangan hanya memfilter tabel tetapi membiarkan summary tetap global.

==================================================
PHASE 17 — PAGINATION
==================================================

Audit pagination Usage Records dan Logs.

Pastikan:

- pagination tidak mengubah total summary.
- summary dihitung dari seluruh dataset yang sesuai filter.
- table hanya mengambil page yang diperlukan.

Jangan menghitung total cost dashboard hanya dari page pertama.

==================================================
PHASE 18 — DATA CONSISTENCY
==================================================

Buat satu real request yang berhasil menggunakan model yang sudah tersedia.

Ambil usage asli dari response.

Contoh:

input = X
output = Y
total = Z

Kemudian ikuti data tersebut sampai:

Usage Record
→ aggregation
→ provider breakdown
→ model breakdown
→ dashboard

Pastikan semua menunjukkan nilai yang sama.

Kemudian hitung cost manual berdasarkan pricing yang benar dan bandingkan dengan sistem.

Jika berbeda:

cari root cause.

Jangan sekadar mengubah angka dashboard.

==================================================
PHASE 19 — MULTIPLE MODEL PRICING TEST
==================================================

Gunakan minimal dua model yang sudah tersedia dan mempunyai pricing berbeda jika tersedia.

Test:

MODEL A
→ request
→ token
→ cost

MODEL B
→ request
→ token
→ cost

Pastikan:

cost A menggunakan pricing A.

cost B menggunakan pricing B.

Jangan menggunakan satu harga global untuk semua model.

==================================================
PHASE 20 — PERSISTENCE
==================================================

Pastikan Usage + Cost tetap benar setelah:

- server restart
- PM2 restart
- application reload

Data tidak boleh kembali ke 0.

Pastikan dashboard mengambil data persistent, bukan memory sementara.

==================================================
PHASE 21 — SECURITY
==================================================

Audit semua Usage/Logs.

Pastikan tidak ada:

- API key
- Authorization header
- provider secret
- password
- private key

yang masuk ke Usage Record atau Dashboard.

Client/API key identifier harus masked.

==================================================
PHASE 22 — TEST SUITE
==================================================

Tambahkan/perbaiki test untuk seluruh jalur:

1. Successful request
2. Token extraction
3. Token normalization
4. Total token calculation
5. Input pricing
6. Output pricing
7. Total cost
8. Different pricing per model
9. Missing usage
10. Missing pricing
11. Blocked request
12. Validation error
13. Upstream error
14. Streaming usage null
15. Streaming usage available
16. Duplicate streaming prevention
17. Provider aggregation
18. Model aggregation
19. Cost aggregation
20. Date filter
21. Provider filter
22. Model filter
23. Pagination
24. Persistence after restart
25. Security/masking

Test kasus penting:

INPUT = 1,000
OUTPUT = 500

Jika pricing:

input = $1 / 1M
output = $2 / 1M

maka:

inputCost = $0.001
outputCost = $0.001
totalCost = $0.002

Gunakan test seperti ini hanya untuk memvalidasi formula, bukan sebagai pricing production.

==================================================
PHASE 23 — REGRESSION
==================================================

Pastikan tidak merusak:

- `/v1/models`
- `/v1/chat/completions`
- `/v1/responses`
- streaming
- Provider Management
- Enable/Disable Provider
- Model Registry
- Usage Tracking
- Usage Logs
- Usage Dashboard
- Backup/Restore

JANGAN memperbaiki failure Model Registry jika failure tersebut tidak berhubungan dengan perubahan Usage.

Fokus pada Usage.

==================================================
PHASE 24 — TEST COMMANDS
==================================================

Jalankan:

npm run lint
npm run build
npm test

Jika test suite mempunyai test provider eksternal/Gorouter yang tidak relevan dengan audit Usage:

jangan mengubah test untuk memalsukan keberhasilan.

Pisahkan:

- Usage tests
- internal tests
- external/provider integration tests

Laporkan dengan jelas.

==================================================
PHASE 25 — ROOT CAUSE, BUKAN PATCH SEMENTARA
==================================================

Jika ditemukan:

- token salah
- total token salah
- cost 0
- cost null
- dashboard tidak update
- aggregation salah
- provider/model tidak konsisten
- usage hilang setelah restart
- streaming tidak tercatat
- pricing salah

jangan hanya patch UI.

Telusuri sampai source data pertama yang salah.

Perbaiki root cause.

==================================================
HASIL AKHIR WAJIB
==================================================

Setelah selesai berikan laporan:

1. ROOT CAUSE masalah Usage.
2. Jalur Usage sebelum perbaikan.
3. Jalur Usage setelah perbaikan.
4. Sumber token.
5. Normalisasi token.
6. Formula total token.
7. Sumber pricing.
8. Formula cost.
9. Currency.
10. Schema Usage yang digunakan.
11. Cost yang disimpan.
12. Aggregation provider.
13. Aggregation model.
14. Dashboard.
15. Filter.
16. Pagination.
17. Streaming.
18. Persistence.
19. Security audit.
20. File yang diubah.
21. Test yang ditambahkan.
22. npm run lint.
23. npm run build.
24. npm test.
25. Jumlah pass/fail/skip.
26. Masalah yang masih tersisa.

PENTING:

Jangan menyentuh Model Registry tanpa alasan yang berhubungan langsung dengan Usage.

Jangan membuat model/provider dummy.

Jangan mengarang token.

Jangan mengarang pricing.

Jangan menghitung cost menggunakan satu harga global jika pricing model berbeda.

Jangan menghitung cost dari total token global jika input/output mempunyai harga berbeda.

Jangan menggunakan floating point yang menyebabkan kesalahan nilai uang.

Jangan menyimpan secret.

Jangan melakukan refactor besar yang tidak diperlukan.

Fokus:
TOKEN → PRICING → COST → STORAGE → AGGREGATION → DASHBOARD.

Pastikan setelah implementasi, satu real request dapat ditelusuri secara penuh dari response provider sampai angka harga/cost yang muncul di dashboard.


```
# Prompt: Usage Tracking & Dashboard Root Cause Fix
```

Lakukan ROOT-CAUSE AUDIT dan perbaikan pada fitur Usage Tracking / Usage Dashboard project nvidia-api.

KONDISI SAAT INI:
- Project sudah deployed di VPS production.
- Provider/model routing sudah terbukti bekerja.
- `mimo-v2.5:free` berhasil menghasilkan response nyata.
- `qwen3-8-max-free` juga berhasil menghasilkan response; timeout sebelumnya adalah upstream intermittent, bukan masalah model registry.
- JANGAN mengubah Model Registry atau Provider Registry kecuali audit membuktikan ada hubungan langsung dengan failure Usage.
- Fokus utama sekarang adalah Usage Tracking, Usage Dashboard, Logs, Responses, dan Streaming.

HASIL TEST TERAKHIR:
Test Files: 8 failed | 27 passed | 1 skipped
Tests: 11 failed | 444 passed | 20 skipped

Failure yang terlihat:
1. tests/responses.test.ts
   POST /v1/responses
   should return responses-compatible structure on success

2. tests/usage-dashboard.test.ts
   GET /admin/logs
   should return blocked requests logs with error details

3. tests/usage-tracking.test.ts
   Request tracking
   should record a blocked request when model has no provider

4. tests/usage-tracking.test.ts
   Request tracking
   should keep recording usage even after many requests

5. tests/stream.test.ts
   POST /v1/chat/completions (streaming)
   should return SSE formatted response
   Error: Stream request timeout

Ada juga failure lain dalam test suite. JANGAN menebak. Audit semua 11 failure.

TUJUAN:
Cari ROOT CAUSE, bukan sekadar membuat test hijau.

LANGKAH WAJIB:

1. BACA FAILURE LENGKAP
- Jalankan hanya test file yang relevan terlebih dahulu.
- Jangan langsung menjalankan full `npm test`.
- Ambil stack trace dan assertion lengkap.
- Kelompokkan failure berdasarkan root cause.

2. AUDIT USAGE TRACKING
Periksa seluruh alur:

incoming request
→ authentication
→ model/provider resolution
→ blocked/validation/success/error
→ provider request
→ response
→ usage extraction
→ recordUsageFor()
→ persistence
→ admin aggregation
→ logs

Pastikan setiap status diperlakukan benar:
- success
- blocked
- validation error
- upstream error
- timeout

3. BLOCKED REQUEST

Khusus kasus:
"should record a blocked request when model has no provider"

Pastikan request yang diblokir karena model tidak memiliki provider:
- tidak diteruskan ke upstream
- menghasilkan HTTP/error status yang sesuai existing behavior
- tetap dicatat sebagai blocked/error sesuai schema existing
- provider/model tetap dicatat jika informasinya tersedia
- error details tersedia pada `/admin/logs`
- tidak tercatat sebagai successful request
- tidak mengarang token usage

Jangan mengubah semantics API hanya agar test lulus.

4. MANY REQUESTS

Khusus:
"should keep recording usage even after many requests"

Audit:
- storage append/write
- race condition
- async logging
- queue/buffer
- singleton state
- database connection
- array truncation
- pagination limit
- ID collision
- counter reset
- error swallowing

Pastikan logging 100+ request atau jumlah yang digunakan test tetap menghasilkan record yang benar.

Jangan membuat hardcoded limit hanya agar test lulus.

5. ADMIN LOGS

Khusus:
GET /admin/logs

Pastikan blocked request memiliki:
- timestamp
- provider jika diketahui
- model
- status
- HTTP status
- error message/details
- request ID jika tersedia

Jangan expose:
- API key
- Authorization header
- provider credential
- secret.

6. `/v1/responses`

Audit endpoint `/v1/responses`.

Pastikan response mengikuti struktur yang memang diharapkan project/API compatibility layer.

Jangan membuat response dummy.

Pastikan:
- valid model request
- authentication
- provider resolution
- upstream request
- response mapping
- usage mapping
- error mapping
- logging

semuanya konsisten dengan `/v1/chat/completions`.

7. STREAMING

Khusus failure:
POST /v1/chat/completions (streaming)
should return SSE formatted response
Error: Stream request timeout

Audit apakah timeout berasal dari:
- test server startup
- request handler
- provider mock/test server
- stream writer
- response headers
- SSE formatting
- usage logger
- socket lifecycle
- request timeout
- cleanup/close behavior.

JANGAN sekadar menaikkan timeout.

Pastikan:
- Content-Type SSE benar
- data chunks dikirim
- `[DONE]`/terminator sesuai format existing
- stream response selesai
- usage logging tidak memblokir stream
- request tidak menggantung
- socket ditutup dengan benar.

8. COMPARE SUCCESSFUL REAL REQUEST

Gunakan hasil real request yang sudah terbukti bekerja sebagai referensi behavior.

Jangan mengganti provider/model yang sedang digunakan.

Pastikan real request menghasilkan:
- response
- usage
- log
- provider
- model
- latency
- status

secara konsisten.

9. ROOT CAUSE FIRST

Sebelum mengubah kode:
- identifikasi file penyebab
- identifikasi fungsi penyebab
- jelaskan root cause
- tentukan apakah beberapa failure berasal dari satu bug shared.

Jangan melakukan banyak perubahan terpisah jika satu root cause dapat memperbaiki beberapa test.

10. FIX

Perbaiki root cause dengan perubahan minimal.

JANGAN:
- membuat mock provider baru
- membuat fake usage
- menghardcode test result
- skip test
- menghapus assertion
- mengubah test hanya agar pass
- menambah timeout secara sembarangan
- mengubah Model Registry tanpa alasan
- refactor besar.

11. REGRESSION

Setelah fix:
jalankan test file yang sebelumnya gagal secara individual.

Kemudian:
- npm run lint
- npm run build

Jika semua test yang relevan sudah lulus, baru jalankan full npm test.

12. PRODUCTION SAFETY

Project sedang deployed di VPS.

Jangan:
- menghapus database
- reset usage data
- reset provider state
- menghapus logs
- mengubah API key
- melakukan destructive migration
- restart production berulang kali tanpa kebutuhan.

Jika perlu restart untuk verification, lakukan satu kali setelah perubahan final.

13. HASIL AKHIR

Laporkan:

ROOT CAUSE:
- failure
- penyebab
- file/fungsi

FIX:
- file yang diubah
- perubahan yang dilakukan

VALIDATION:
- responses.test.ts
- usage-dashboard.test.ts
- usage-tracking.test.ts
- stream.test.ts
- test lainnya yang terkait

Kemudian:
- lint
- build
- full test

Juga laporkan:
- jumlah pass/fail/skip
- apakah Model Registry tetap tidak berubah
- apakah real model request tetap bekerja.

PENTING:
Jangan push/commit dulu.
Jangan mengubah Model Registry.
Jangan mengubah provider routing.
Fokus hanya pada root cause Usage/Logs/Responses/Streaming.

```

# Prompt: Final Cost Verification & Runtime Consistency Audit
```
Lakukan FINAL VERIFICATION pada project `nvidia-api`.

JANGAN langsung mengubah source code.

Kondisi saat ini:
- npm run lint = PASS
- npm run build = PASS
- npm test = 455 passed, 20 skipped, 0 failed
- usage-store.ts sudah memiliki historical cost backfill
- cost-display.test.ts sudah diperbarui
- beberapa record Gorouter/Claude Opus 4.8 masih perlu diverifikasi costUsd-nya di runtime/admin dashboard.

TUJUAN:
Pastikan costUsd yang dihitung oleh aplikasi benar-benar sama dengan data Usage yang tersimpan dan yang ditampilkan Admin Dashboard.

1. AUDIT PRICING

Cari exact pricing entry untuk:

provider:
gorouter

model:
claude-opus-4-8

Pastikan pricing yang digunakan runtime adalah:

input = $5 / 1M tokens
output = $25 / 1M tokens

Jangan membuat pricing kedua/duplikat.

2. VERIFIKASI RECORD NYATA

Baca data Usage yang benar-benar digunakan runtime.

Cari minimal beberapa record:

provider = gorouter
model = claude-opus-4-8

Untuk setiap record catat:

promptTokens
completionTokens
totalTokens
costUsd

JANGAN mengubah record hanya untuk debugging.

3. HITUNG MANUAL

Untuk setiap record dengan costUsd null, hitung:

cost =
(promptTokens * 5 / 1,000,000)
+
(completionTokens * 25 / 1,000,000)

Bandingkan dengan hasil `costForRecord()` / `computeCostUsd()` yang sebenarnya digunakan aplikasi.

Jangan membulatkan nilai internal.

4. CONTOH DATA

Untuk:

promptTokens = 88183
completionTokens = 234

hasil yang benar adalah:

88183 * 5 / 1,000,000
+
234 * 25 / 1,000,000

= 0.447835 USD

Verifikasi apakah aplikasi menghasilkan angka yang sama.

5. HISTORICAL BACKFILL

Pastikan record lama dengan:

costUsd = null

dapat diperkaya secara in-memory sesuai behavior yang sudah dibuat.

PENTING:
- Jangan mengubah token.
- Jangan mengubah provider.
- Jangan mengubah model.
- Jangan mengubah timestamp.
- Jangan mengubah totalTokens.
- Jangan menimpa costUsd yang sudah valid.
- Jangan menulis hasil backfill ke disk jika arsitektur existing memang sengaja read-only/in-memory.

Pastikan tidak ada fake `$0`.

6. ADMIN DASHBOARD

Periksa langsung endpoint/page yang digunakan:

/admin/usage
/admin/usage/providers
/admin/usage/models
/admin/usage/records
/admin/logs

Pastikan cost yang ditampilkan berasal dari data usage yang sama.

Periksa:

- total estimated cost
- provider cost
- model cost
- individual record cost

Pastikan tidak ada perbedaan antara API dan UI.

7. STORAGE CONSISTENCY

Audit:

- lokasi `usage-records.json`
- file/database/storage yang sebenarnya digunakan runtime
- working directory process
- environment/configuration yang menentukan storage path

Pastikan aplikasi tidak membaca file Usage yang berbeda dari file yang sedang diaudit.

Jangan membuat storage baru.

8. PROCESS CONSISTENCY

Periksa proses Node/PM2 yang menjalankan `nvidia-api`.

Pastikan hanya instance yang memang diperlukan yang melayani port aplikasi.

Jika ditemukan dua server process yang menggunakan storage/port berbeda:

JANGAN langsung kill process.

Identifikasi:
- PID
- command
- working directory
- port
- PM2 process
- environment/storage path

Laporkan apakah ada risiko dashboard membaca instance/storage yang berbeda.

Jangan melakukan destructive action.

9. RUNTIME RESTART TEST

Jika aman:

- restart hanya instance `nvidia-api` yang memang digunakan production.
- jangan menghapus Usage data.
- setelah restart cek kembali:
  - usage count
  - gorouter/claude-opus-4-8 records
  - costUsd
  - admin dashboard

Pastikan behavior tetap konsisten.

10. NO CODE CHANGE IF ALREADY CORRECT

Jika seluruh hasil benar:

JANGAN mengubah source code.

Laporkan bahwa implementation sudah benar dan masalah sebelumnya hanya masalah verification/runtime consistency.

Jika ditemukan bug nyata:
- ubah hanya bagian minimal yang diperlukan.
- jangan refactor besar.
- jalankan ulang lint/build/test.

11. FINAL TEST

Jika ada perubahan kode, jalankan:

npm run lint
npm run build
npm test

Target:

0 failed

12. LAPORAN WAJIB

Laporkan tabel:

Record | Provider | Model | Prompt | Completion | Total | Stored costUsd | Recomputed cost

Kemudian laporkan:

- exact pricing yang digunakan
- hasil manual calculation
- hasil `computeCostUsd()`
- hasil historical backfill
- lokasi storage Usage
- process/PM2 yang aktif
- apakah ada duplicate server
- hasil Admin Usage
- hasil Admin Logs
- apakah restart mengubah hasil
- lint
- build
- test

PENTING:
Ini adalah AUDIT/VERIFICATION.

Jangan menambah fitur baru.
Jangan membuat mock data.
Jangan mengarang usage.
Jangan mengubah token.
Jangan menghapus Usage records.
Jangan menghapus process secara sembarangan.
Jangan membuat database/storage baru.


```
# Prompt: Fix Gorouter Cost Calculation
```

PERBAIKI MASALAH COST CALCULATION PADA PROJECT `nvidia-api`.

HASIL AUDIT:
Usage record nyata sudah memiliki:
- provider = `gorouter`
- model = `claude-opus-4-8`
- promptTokens tersedia
- completionTokens tersedia
- totalTokens tersedia
- tetapi costUsd = null

PENYEBAB:
`src/lib/pricing.ts` melakukan exact lookup berdasarkan:
provider + model

Saat ini belum ada pricing key:
`gorouter/claude-opus-4-8`

Akibatnya model price tidak ditemukan dan costUsd menjadi null.

TUJUAN:
Buat cost calculation untuk Gorouter bekerja berdasarkan exact provider + model tanpa merusak model/provider lain.

1. PRICING

Tambahkan entry:

`gorouter/claude-opus-4-8`

dengan:
- inputPerM = 5
- outputPerM = 25

Harga tersebut adalah harga list standar Claude Opus 4.8:
$5 per 1M input tokens
$25 per 1M output tokens.

2. JANGAN MENGUBAH ARSITEKTUR

Pertahankan:
- `PRICING_REGISTRY`
- `getModelPrice()`
- `computeCostUsd()`
- `costForRecord()`

Jangan membuat pricing system kedua.

3. PROVIDER LAIN

Jangan memberikan default price global untuk semua model Gorouter.

Pricing harus tetap exact:
`provider/model`

Jangan membuat:
`gorouter/*`

Karena model berbeda dapat memiliki harga berbeda.

4. HISTORICAL USAGE RECORDS

PENTING:

Record lama seperti:

provider = gorouter
model = claude-opus-4-8
costUsd = null

harus dapat dihitung ulang menggunakan pricing baru jika:
- token tersedia
- costUsd masih null

Jangan mengubah record yang sudah memiliki costUsd valid.

Jangan mengubah:
- promptTokens
- completionTokens
- totalTokens
- timestamp
- provider
- model

Hanya hitung costUsd jika memang sebelumnya belum tersedia.

5. COST FORMULA

Gunakan:

(promptTokens * inputPerM / 1,000,000)
+
(completionTokens * outputPerM / 1,000,000)

Jangan menggunakan totalTokens sebagai input price.

6. VALIDASI CONTOH

Untuk record:

promptTokens = 88,183
completionTokens = 234

cost:

(88183 × 5 / 1,000,000)
+
(234 × 25 / 1,000,000)

Harus menghasilkan:

$0.447835

Jangan membulatkan nilai yang disimpan di UsageRecord.
Pembulatan hanya pada tampilan UI.

7. DASHBOARD

Pastikan hasil cost muncul konsisten pada:

- `/admin/usage`
- `/admin/usage/providers`
- `/admin/usage/models`
- `/admin/usage/records`
- `/admin/logs`

Total cost juga harus ikut terakumulasi dengan benar.

8. TEST

Tambahkan test khusus:

- gorouter + claude-opus-4-8 menemukan pricing
- cost calculation dengan token nyata
- cost zero tetap valid jika memang harga $0
- unknown provider/model tetap menghasilkan null
- historical record dengan costUsd null dapat dihitung ulang
- record dengan costUsd yang sudah ada tidak ditimpa
- input/output token tidak berubah
- provider/model tidak berubah

9. REGRESSION

Pastikan pricing yang sudah ada tetap bekerja.

Jalankan:

npm run lint
npm run build
npm test

Jangan mengubah test hanya agar lulus.

10. JANGAN

- Jangan membuat mock usage.
- Jangan mengarang token.
- Jangan mengubah token yang tersimpan.
- Jangan mengubah provider/model.
- Jangan membuat default pricing untuk semua Gorouter.
- Jangan membuat database baru.
- Jangan refactor besar.
- Jangan menghapus data Usage.
- Jangan menghapus historical records.

SETELAH SELESAI:

Laporkan:
- file yang diubah
- pricing Gorouter yang ditambahkan
- hasil perhitungan contoh
- apakah historical costUsd null berhasil di-backfill
- hasil `/admin/usage`
- hasil `/admin/usage/providers`
- hasil `/admin/usage/models`
- lint
- build
- test
- masalah yang masih tersisa.

```
# Real Cost Calculation
```
Prompt: Real Usage Cost Calculation

Lanjutkan implementasi project `nvidia-api`.

KONDISI SAAT INI:
- Usage Tracking sudah berjalan.
- Real request sudah menghasilkan usage token.
- Dashboard sudah menampilkan:
  - Total requests
  - Successful
  - Errors
  - Blocked
  - Total tokens
- Saat ini Total Tokens sudah terisi nyata, contoh:
  87,948,548 tokens
- Tetapi `EST. COST (TOTAL)` masih `$0`.

TUJUAN:
Hubungkan usage token nyata dengan pricing engine sehingga cost benar-benar dihitung dan ditampilkan.

JANGAN membuat usage tracking baru.
JANGAN mengestimasi token.
JANGAN membuat angka cost palsu.

1. AUDIT PRICING FLOW

Audit source code existing:

- src/lib/pricing.ts
- src/lib/usage-store.ts
- src/lib/pipeline.ts
- src/services/provider.ts
- src/services/stream-usage.ts
- src/routes/admin.ts
- src/admin/dashboard.ts
- file lain yang benar-benar terlibat dalam pricing/usage.

Cari penyebab kenapa:

total tokens > 0

tetapi:

estimated cost = $0

Kemungkinan yang harus diaudit:
- pricing lookup tidak menemukan provider/model
- model ID tidak cocok
- provider ID tidak cocok
- pricing hanya menghitung jika harga tersedia
- usage record belum menyimpan cost
- aggregation cost belum menjumlahkan cost
- dashboard membaca field cost yang salah
- input/output token null pada record tertentu
- cost default 0 menutupi pricing lookup failure.

Jangan langsung menebak.
Temukan root cause sebenarnya.

2. REAL PRICING LOOKUP

Pastikan pricing dihitung berdasarkan:

provider
+
exact model ID
+
input tokens
+
output tokens

Gunakan pricing configuration yang sudah ada di project.

Jangan membuat harga provider/model secara sembarangan.

Jika exact model belum memiliki pricing:
- jangan membuat harga palsu
- tampilkan status pricing unavailable/unpriced
- jangan menyamarkan sebagai `$0`.

3. COST FORMULA

Jika harga tersedia:

inputCost =
(inputTokens / pricingUnit) * inputPrice

outputCost =
(outputTokens / pricingUnit) * outputPrice

totalCost =
inputCost + outputCost

Gunakan unit pricing yang benar sesuai struktur pricing existing.

Jangan menganggap harga per token jika konfigurasi sebenarnya per 1K/1M token.

4. REAL USAGE RECORD

Setiap successful request yang memiliki usage nyata harus dapat menghasilkan:

{
  inputTokens,
  outputTokens,
  totalTokens,
  inputCost,
  outputCost,
  totalCost
}

Jika provider tidak memberikan token usage:
- token tetap null sesuai behavior existing
- cost harus null/unpriced jika cost tidak dapat dihitung
- jangan mengestimasi token
- jangan menghasilkan cost palsu.

5. EXISTING RECORD COMPATIBILITY

Jangan merusak usage records lama.

Jika record lama hanya memiliki:

inputTokens
outputTokens
totalTokens

tetapi belum memiliki cost:

sediakan mekanisme safe recalculation/backfill hanya jika aman.

Jangan mengubah historical data secara diam-diam tanpa alasan.

Jika pricing saat ini tersedia untuk provider/model tersebut, cost historical dapat dihitung ulang dari token yang sudah tersimpan.

6. DASHBOARD

Perbaiki:

`/admin/usage`

agar:

EST. COST (TOTAL)

tidak lagi selalu `$0`.

Dashboard harus menghitung dari usage records yang benar-benar memiliki cost.

Tampilkan dengan precision yang cukup untuk nilai kecil.

Jangan membulatkan terlalu awal.

Contoh:
Jika cost sebenarnya:

0.000154

jangan langsung menjadi:

$0

Gunakan precision yang tetap dapat menunjukkan nilai kecil.

7. PROVIDER BREAKDOWN

Pastikan:

`/admin/usage/providers`

menampilkan cost per provider:

Provider
Requests
Input Tokens
Output Tokens
Total Tokens
Estimated Cost

8. MODEL BREAKDOWN

Pastikan:

`/admin/usage/models`

menampilkan:

Provider
Model
Requests
Input Tokens
Output Tokens
Total Tokens
Estimated Cost

Cost harus dihitung dari exact provider + exact model.

9. USAGE RECORDS

Pastikan:

`/admin/usage/records`

menampilkan cost setiap request jika pricing tersedia.

Minimal:

Input Tokens
Output Tokens
Total Tokens
Input Cost
Output Cost
Total Cost

Jika pricing unavailable:
tampilkan `N/A` atau status `Unpriced`, bukan `$0`.

10. PROVIDER/MODEL NORMALIZATION

Audit kemungkinan mismatch seperti:

provider:
`nvidia`

vs

`NVIDIA`

atau model:

`deepseek-ai/deepseek-v4-flash-0731`

vs nama/model alias lainnya.

Pricing lookup harus menggunakan canonical provider/model identifier yang memang digunakan runtime.

Jangan membuat alias sembarangan.

11. REAL COST TEST

Tambahkan test yang menggunakan angka nyata tetapi deterministic.

Contoh:

inputTokens = 1000
outputTokens = 500

Dengan pricing configuration existing:

expectedCost =
inputCost + outputCost

Pastikan hasil exact/precision benar.

Test juga:

- input only
- output only
- input + output
- zero tokens
- null usage
- unknown provider
- unknown model
- pricing unavailable
- very small cost
- large token count
- aggregation multiple records.

12. REGRESSION

Pastikan tidak merusak:

- Provider Management
- Enable/Disable Provider
- Model Registry
- /v1/models
- API request
- streaming
- Usage Tracking
- Usage Logs
- Usage Dashboard
- Backup/Restore.

Jangan melakukan refactor besar.

13. REAL DATA AUDIT

Setelah implementasi, gunakan usage data yang SUDAH ADA.

Jangan menghapus database/usage records.

Hitung ulang aggregate cost dari existing usage records jika memungkinkan.

Pastikan dashboard tidak lagi menampilkan `$0` hanya karena cost calculation belum terhubung.

Bandingkan:

Total token pada dashboard
vs
sum token usage records

dan:

Total cost
vs
sum calculated cost records.

14. PRECISION

Cost harus menggunakan precision aman.

Jangan menggunakan integer untuk cost.

Hindari floating-point rounding terlalu awal.

Jika project sudah memiliki helper Decimal/precision, gunakan helper tersebut.

15. SECURITY

Pastikan pricing/cost implementation tidak menyebabkan:
- API key masuk log
- Authorization header tersimpan
- credential provider terekspos.

16. TESTING

Jalankan:

npm run lint
npm run build
npm test

Jangan menjalankan test/integration test Gorouter.app.

Jangan mengubah test hanya supaya lulus.

17. HASIL AKHIR

Laporkan:

1. Root cause kenapa Est. Cost sebelumnya `$0`.
2. File yang diubah.
3. Pricing lookup yang digunakan.
4. Formula cost.
5. Apakah cost tersimpan pada usage record.
6. Apakah dashboard total cost sudah benar.
7. Provider breakdown cost.
8. Model breakdown cost.
9. Historical usage apakah berhasil dihitung.
10. Pricing unavailable behavior.
11. Precision/rounding behavior.
12. Test yang ditambahkan.
13. npm run lint.
14. npm run build.
15. npm test.
16. Jumlah pass/fail/skip.
17. Regression yang ditemukan.

PENTING:
- Gunakan token usage NYATA yang sudah tersimpan.
- Jangan mengarang token.
- Jangan mengarang harga.
- Jangan membuat provider/model dummy.
- Jangan menghapus usage lama.
- Jangan membuat `$0` sebagai fallback ketika pricing sebenarnya tidak ditemukan.
- Jika pricing tidak tersedia, tampilkan `N/A/Unpriced` agar masalah terlihat jelas.
- Jangan test Gorouter.app.


```
# Prompt — Real Usage & Cost Verification
```

Lanjutkan implementasi pricing pada project nvidia-api sampai REAL VERIFICATION.

TUJUAN:
Pastikan setiap request nyata menghitung biaya USD berdasarkan token request tersebut, bukan token kumulatif dari OpenCode/session.

1. SIAPKAN ENVIRONMENT
- Jika node_modules belum ada, install dependency yang diperlukan.
- Jangan mengubah source hanya karena dependency belum tersedia.
- Jalankan lint/build/test jika environment sudah siap.

2. AUDIT PRICING
Pastikan cost hanya berasal dari:
promptTokens + completionTokens
pada request tersebut.

JANGAN:
- membaca token kumulatif OpenCode
- menjumlahkan usage session
- menghitung token dari request sebelumnya
- double counting
- menggunakan estimasi token jika upstream menyediakan usage asli.

3. REAL REQUEST TEST
Jalankan server VPS2 dan lakukan minimal 3 request nyata menggunakan provider/model yang memang tersedia.

Untuk setiap request catat:
- request ID
- provider
- model
- prompt tokens
- completion tokens
- total tokens
- cost USD
- latency

Validasi:
totalTokens = promptTokens + completionTokens

4. COST VALIDATION
Gunakan pricing configuration yang sudah dibuat.

Untuk setiap request hitung secara independen:

cost = (promptTokens × inputPricePerToken)
     + (completionTokens × outputPricePerToken)

Bandingkan hasil perhitungan manual dengan cost yang disimpan server.

Harus sama sesuai precision/rounding yang ditentukan.

5. DASHBOARD
Buka /admin dan pastikan:
- total tokens berasal dari seluruh record request
- estimated cost USD adalah SUM cost masing-masing record
- refresh halaman tidak menambah usage/cost
- tidak ada token kumulatif OpenCode yang ikut dihitung.

6. LOGS
Buka /admin/logs.

Setiap request sukses harus menampilkan:
- prompt tokens
- completion tokens
- total tokens
- cost USD

Contoh format:
Tokens: 3,000
Cost: $0.001234

Cost harus milik request tersebut, bukan total kumulatif.

7. ANTI DOUBLE-COUNTING
Lakukan:
request A → catat token + cost
request B → catat token + cost

Pastikan:
cost(A+B) = cost(A) + cost(B)

Refresh dashboard/logs berkali-kali tidak boleh mengubah angka.

8. PERSISTENCE
Restart server.

Setelah restart:
- record tetap ada
- token tetap sama
- cost tetap sama
- aggregate dashboard tetap sama.

9. SECURITY
Pastikan pricing/usage tidak menyimpan:
- API key
- Authorization header
- provider secret
- credential OpenCode.

10. HASIL AKHIR
Jangan berhenti pada unit test.

WAJIB lakukan REAL REQUEST dan tampilkan laporan:

Request #1:
provider =
model =
promptTokens =
completionTokens =
totalTokens =
costUsd =

Request #2:
...

Request #3:
...

Kemudian tampilkan:
- total request
- total prompt tokens
- total completion tokens
- total tokens
- total cost USD
- apakah dashboard sesuai
- apakah logs sesuai
- apakah refresh aman
- apakah restart aman
- lint/build/test result.

Jika ada perbedaan angka, JANGAN menutupinya. Cari sumber double counting/cumulative usage dan perbaiki penyebab sebenarnya.

```
# 
```

Lakukan implementasi FINAL untuk fitur PRICING / ESTIMATED COST pada project `nvidia-api`.

KONDISI PENTING:
- Saya sedang mengerjakan project ini di VPS2.
- VPS1 menjalankan production `nvidia-api` dan JANGAN disentuh.
- Jangan melakukan deploy, restart, kill process, SSH, atau perubahan apa pun ke VPS1.
- Semua perubahan dan testing hanya di VPS2.
- Jangan menghapus data existing.
- Jangan melakukan refactor besar.
- Jangan mengubah sistem token accounting yang sudah benar.
- Fokus pada pricing USD berdasarkan TOKEN PADA REQUEST INDIVIDUAL.

TUJUAN UTAMA:

Saya ingin setiap request memiliki:

prompt_tokens
completion_tokens
total_tokens
cost_usd

Contoh satu request:

prompt_tokens = 120000
completion_tokens = 220
total_tokens = 120220
cost_usd = $0.123456

Nilai cost harus dihitung HANYA dari token request tersebut.

JANGAN:
- mengambil token kumulatif dari OpenCode/session
- menjumlahkan token dari request sebelumnya
- menghitung ulang token berdasarkan dashboard
- menggunakan cumulative/session token sebagai usage request
- melakukan double counting
- mengestimasi token jika upstream tidak memberikan usage

==================================================
1. AUDIT IMPLEMENTASI EXISTING TERLEBIH DAHULU
==================================================

Sebelum mengubah kode:

Audit source code yang berhubungan dengan:

- usage
- usage-store
- provider
- provider pricing
- model pricing
- response parsing
- extractUsageFromResult
- streaming usage
- dashboard
- logs
- `/admin/usage`
- `/admin/logs`
- cost/price jika sudah ada

Cari terutama:

- `src/lib/usage-store.ts`
- `src/lib/pricing.ts`
- `src/services/provider.ts`
- `src/services/team-usage.ts`
- `src/providers/agentrouter/response.ts`
- `src/admin/dashboard.ts`
- `src/admin/index.html`
- seluruh code yang menghitung `promptTokens`
- seluruh code yang menghitung `completionTokens`
- seluruh code yang menghitung `totalTokens`

Jangan langsung mengubah code.

Pertama pahami alur:

UPSTREAM RESPONSE
→ extract usage
→ individual usage record
→ pricing
→ cost_usd
→ Logs
→ Dashboard aggregation

Pastikan pricing ditempatkan pada titik yang tepat.

==================================================
2. SUMBER TOKEN WAJIB INDIVIDUAL REQUEST
==================================================

Untuk setiap request:

prompt_tokens
completion_tokens
total_tokens

harus berasal dari usage response request tersebut.

Jika upstream memberikan:

usage.prompt_tokens
usage.completion_tokens
usage.total_tokens

gunakan nilai tersebut.

Validasi:

total_tokens === prompt_tokens + completion_tokens

Jika provider memberikan total_tokens yang berbeda:
- jangan diam-diam mengubah nilai
- simpan nilai upstream sesuai policy existing
- tampilkan discrepancy hanya jika diperlukan untuk debugging
- jangan melakukan double counting.

PENTING:

Jangan pernah menggunakan:
- session token
- cumulative token
- OpenCode accumulated token
- dashboard total
- previous request usage

untuk menentukan `cost_usd` sebuah request.

==================================================
3. STRUKTUR COST PER REQUEST
==================================================

Tambahkan field persistent:

`cost_usd`

pada usage record jika belum ada.

Idealnya setiap record memiliki:

{
  promptTokens,
  completionTokens,
  totalTokens,
  costUsd
}

Gunakan naming convention existing jika project sudah mempunyai field pricing yang berbeda.

Cost harus disimpan sebagai nilai numerik yang aman untuk perhitungan.

Jangan menggunakan string seperti:

"$0.001"

untuk storage.

Storage harus berupa number.

UI boleh menampilkan:

`$0.001000`

==================================================
4. PRICING MODEL
==================================================

Buat pricing berdasarkan:

provider + model

Contoh konsep:

{
  provider: "tokenharbor",
  model: "deepseek-v4-flash:free",
  inputPricePer1M: 0,
  outputPricePer1M: 0
}

Formula:

inputCost =
(prompt_tokens / 1,000,000) * inputPricePer1M

outputCost =
(completion_tokens / 1,000,000) * outputPricePer1M

costUsd =
inputCost + outputCost

Jangan menggunakan total_tokens langsung jika pricing input/output berbeda.

Jika pricing model gratis:

inputPricePer1M = 0
outputPricePer1M = 0

maka:

costUsd = 0

==================================================
5. PRICING HARUS MODEL-SPECIFIC
==================================================

Jangan menggunakan satu harga global untuk semua model.

Pricing harus dapat dibedakan:

provider
→ model
→ input price
→ output price

Contoh:

provider A / model X
input = ...
output = ...

provider A / model Y
input = ...
output = ...

provider B / model X
input = ...
output = ...

Jika pricing belum tersedia untuk suatu provider/model:

JANGAN mengarang harga.

Gunakan:

costUsd = null

dan tandai sebagai:

pricing unavailable

Jangan menganggap model gratis hanya karena harga belum dikonfigurasi.

==================================================
6. PRESISI USD
==================================================

Jangan membulatkan cost terlalu awal.

Contoh:

cost internal:
0.000873421

UI:
$0.000873

atau gunakan precision yang konsisten.

Dashboard harus menggunakan nilai internal penuh untuk SUM.

Jangan:

request 1 → round
request 2 → round
request 3 → round
kemudian SUM

Lebih aman:

SUM raw cost
→ format untuk display.

==================================================
7. LOGS
==================================================

Pada halaman `/admin/logs`, tambahkan kolom:

COST

Contoh:

PROMPT     COMPLETION     TOTAL       COST
120,895    220            121,115     $0.001234

Pastikan cost berasal dari record request tersebut.

Jika token null:

PROMPT = —
COMPLETION = —
TOTAL = —
COST = —

atau `$0.00` hanya jika pricing/token policy existing memang mendefinisikan demikian.

Jangan menghitung cost dari dashboard.

==================================================
8. DASHBOARD
==================================================

Pada `/admin` / Overview tambahkan:

EST. COST (TOTAL)

Contoh:

$0.123456

Nilai ini harus:

SUM(costUsd dari individual usage records)

BUKAN:

harga berdasarkan total token dashboard secara terpisah.

Tujuannya mencegah perbedaan:

SUM(request cost)
vs
calculate(total dashboard tokens)

Keduanya secara matematis bisa berbeda jika:
- pricing berbeda antar model
- provider berbeda
- model berbeda
- sebagian request pricing unavailable

Karena itu source of truth:

`SUM(individual record.costUsd)`

==================================================
9. PROVIDER USAGE
==================================================

Pada Provider Usage:

tampilkan:

Provider
Requests
Prompt Tokens
Completion Tokens
Total Tokens
Cost USD

Contoh:

TokenHarbor
Requests: 100
Prompt: 1,000,000
Completion: 20,000
Total: 1,020,000
Cost: $0.123456

Cost harus merupakan:

SUM(costUsd WHERE provider = provider)

==================================================
10. MODEL USAGE
==================================================

Pada Model Usage:

tampilkan:

Provider
Model
Requests
Prompt Tokens
Completion Tokens
Total Tokens
Cost USD

Cost:

SUM(individual costUsd)

berdasarkan:

provider + model

==================================================
11. FILTER DAN LOGS
==================================================

Jika Logs difilter berdasarkan:

provider
model
status
date range

cost harus ikut mengikuti filter.

Contoh:

All:
$10.00

Provider NVIDIA:
$3.00

Provider TokenHarbor:
$7.00

Jangan menampilkan total global ketika user sedang memfilter record.

==================================================
12. ERROR / BLOCKED REQUEST
==================================================

Request:

SUCCESS
→ boleh memiliki token + cost jika usage tersedia.

UPSTREAM ERROR
→ gunakan usage hanya jika upstream benar-benar memberikannya.

BLOCKED
→ biasanya:

promptTokens = null
completionTokens = null
totalTokens = null
costUsd = null

Jangan membebankan biaya kepada request yang tidak pernah diteruskan upstream.

INVALID MODEL
→ tidak boleh dihitung sebagai paid inference.

PROVIDER DISABLED
→ tidak boleh dihitung sebagai paid inference.

==================================================
13. STREAMING
==================================================

Untuk streaming:

Jika final SSE chunk memberikan usage:

gunakan usage tersebut.

Jika usage tersedia:

promptTokens
completionTokens
totalTokens
costUsd

dicatat.

Jika upstream streaming memberikan:

usage: null

maka:

promptTokens = null
completionTokens = null
totalTokens = null
costUsd = null

JANGAN:
- mengambil cumulative session token
- membaca token dari OpenCode
- menjumlahkan request sebelumnya
- melakukan estimasi
- membuat cost palsu

Streaming tidak boleh rusak hanya karena pricing.

==================================================
14. ANTI DOUBLE COUNTING
==================================================

Ini WAJIB.

Satu request hanya boleh menghasilkan:

SATU usage record.

Pastikan tidak ada:

upstream usage
+
stream usage
+
dashboard usage

yang disimpan sebagai tiga record.

Pastikan:

request ID / trace ID

digunakan jika tersedia untuk memastikan satu request tidak dicatat dua kali.

Jika `recordUsageFor()` dipanggil lebih dari sekali untuk lifecycle request yang sama:

gunakan mekanisme existing atau tambahkan guard yang aman agar tidak terjadi duplicate usage.

Jangan mengubah behavior request utama.

==================================================
15. ANTI CUMULATIVE OPENCODE TOKEN
==================================================

Audit seluruh code pricing dan usage.

Pastikan tidak ada pola seperti:

previousTokens + currentTokens

atau:

sessionTokens

atau:

cumulativeUsage

yang digunakan sebagai token individual request.

Contoh:

Request #1:
100,000 prompt
200 completion
cost dihitung dari 100,200

Request #2:
80,000 prompt
300 completion
cost dihitung dari 80,300

Request #2 TIDAK BOLEH menjadi:

180,500

karena token request #1 sudah pernah dihitung.

Setiap request berdiri sendiri.

==================================================
16. PRICING CONFIGURATION
==================================================

Buat pricing configuration yang mudah diperluas.

Contoh struktur:

pricing:
  provider
    model
      inputPricePer1M
      outputPricePer1M
      currency

Gunakan struktur project yang paling sesuai.

Jangan hardcode pricing di UI.

UI harus membaca cost dari backend.

Jika project sudah memiliki `src/lib/pricing.ts`:
gunakan dan perbaiki file tersebut daripada membuat pricing system kedua.

Jangan membuat dua sumber pricing.

==================================================
17. CURRENCY
==================================================

Currency internal:

USD

UI:

`$0.001234`

Jangan menggunakan kurs Rupiah.

Jangan melakukan konversi IDR.

Pricing provider diasumsikan USD kecuali konfigurasi secara eksplisit menyatakan currency lain.

==================================================
18. API / BACKEND
==================================================

Pastikan endpoint admin mengembalikan:

costUsd

pada:

- usage summary
- provider usage
- model usage
- usage records
- logs

Jangan menghitung cost di frontend berdasarkan token.

Frontend hanya menampilkan nilai cost dari backend.

Ini penting supaya:

Dashboard
Logs
Provider Usage
Model Usage

menggunakan source of truth yang sama.

==================================================
19. TEST WAJIB
==================================================

Tambahkan regression test untuk:

TEST 1:
Request A:
1000 prompt
500 completion

pricing:
input = $1 / 1M
output = $2 / 1M

Expected:

cost =
(1000 / 1M * 1)
+
(500 / 1M * 2)

TEST 2:
Request B:
2000 prompt
100 completion

Pastikan cost B dihitung hanya dari B.

TEST 3:
A + B

Pastikan:

dashboard cost
=
cost A + cost B

TEST 4:
Simulasikan cumulative/session token.

Pastikan cumulative token TIDAK mempengaruhi cost individual.

TEST 5:
Streaming usage null.

Expected:

costUsd = null

TEST 6:
Streaming usage tersedia.

Expected:

costUsd dihitung dari usage streaming tersebut.

TEST 7:
Provider disabled.

Expected:

costUsd = null

TEST 8:
Invalid model.

Expected:

costUsd = null

TEST 9:
Pricing unavailable.

Expected:

costUsd = null

TEST 10:
Free model.

Expected:

costUsd = 0

TEST 11:
Duplicate `recordUsageFor()` untuk request ID yang sama.

Expected:

hanya satu usage record.

TEST 12:
Dashboard aggregation.

Pastikan:

SUM(record.costUsd)
=
dashboard total cost

TEST 13:
Provider aggregation.

Pastikan:

SUM(cost per provider)
=
global cost

TEST 14:
Model aggregation.

Pastikan:

SUM(cost per model)
=
global cost

==================================================
20. SECURITY
==================================================

Pricing implementation tidak boleh:
- menyimpan API key
- menyimpan Authorization header
- membocorkan credential
- menyimpan secret
- memasukkan secret ke backup

Audit juga backup supaya `costUsd` boleh masuk backup tetapi credential tetap tidak.

==================================================
21. BUILD & VALIDATION
==================================================

Setelah implementasi:

jalankan jika environment memungkinkan:

npm run lint
npm run build
npm test

Jika dependency/environment VPS2 belum lengkap:

- jangan mengarang hasil
- laporkan command yang gagal
- jelaskan bahwa failure disebabkan environment
- tetap lakukan static/code audit yang memungkinkan.

JANGAN menyentuh VPS1.

JANGAN restart VPS1.

==================================================
22. FINAL VERIFICATION
==================================================

Buat test nyata/internal dengan minimal 3 request:

Request #1:
usage individual tertentu

Request #2:
usage individual berbeda

Request #3:
usage individual berbeda

Verifikasi:

Logs:
request #1 → cost #1
request #2 → cost #2
request #3 → cost #3

Dashboard:

cost total
=
cost #1 + cost #2 + cost #3

Kemudian refresh dashboard.

Pastikan angka tidak berubah hanya karena refresh.

Kemudian buat request ke model/provider berbeda jika tersedia.

Pastikan pricing mengikuti provider + model.

==================================================
23. ACCEPTANCE CRITERIA
==================================================

IMPLEMENTASI DIANGGAP BERHASIL JIKA:

1. Setiap request memiliki cost individual jika pricing + usage tersedia.
2. Cost tidak berasal dari token kumulatif OpenCode.
3. Tidak ada double counting.
4. Logs menampilkan cost per request.
5. Dashboard menampilkan total cost.
6. Provider Usage menampilkan cost provider.
7. Model Usage menampilkan cost model.
8. Filter Logs memfilter cost dengan benar.
9. Streaming usage null tidak menghasilkan cost palsu.
10. Provider disabled tidak menghasilkan cost.
11. Invalid model tidak menghasilkan cost.
12. Pricing unavailable menghasilkan null.
13. Free model menghasilkan $0.
14. Dashboard total = SUM individual request cost.
15. Pricing input/output dapat berbeda per model.
16. Semua cost menggunakan USD.
17. Frontend tidak menghitung ulang pricing.
18. Tidak ada credential/secret yang bocor.
19. Existing token accounting tidak rusak.
20. VPS1 TIDAK disentuh.

==================================================
HASIL AKHIR
==================================================

Laporkan secara ringkas:

- File yang diubah
- Source of truth pricing
- Struktur pricing
- Formula cost
- Contoh cost request
- Status Logs
- Status Dashboard
- Provider Usage
- Model Usage
- Streaming
- Anti cumulative token
- Anti double counting
- Test result
- lint result
- build result
- masalah environment jika ada

PENTING TERAKHIR:

JANGAN:
- menyentuh VPS1
- restart VPS1
- mengubah production VPS1
- mengambil token cumulative OpenCode
- menghitung cost dari dashboard token
- membuat pricing kedua/duplikat
- mengarang harga model
- mengarang token
- mengestimasi token
- membuat mock provider
- menghapus data existing
- melakukan refactor besar.

Fokus hanya pada pricing USD yang aman, akurat, per-request, dan tidak kumulatif.

```
# 
```


Implementasikan dan perbaiki fitur TOKEN COST / USD COST pada project `nvidia-api`.

MASALAH SAAT INI:
Dashboard sudah memiliki card:

EST. COST (TOTAL)
$0

Tetapi cost selalu `$0`.

Yang diinginkan:
- setiap request memiliki cost sendiri berdasarkan token request tersebut
- Logs menampilkan cost per request
- Dashboard menjumlahkan cost dari seluruh request individual
- jangan mengambil token kumulatif/session dari OpenCode
- jangan menghitung ulang berdasarkan hasil token yang terus bertambah
- jangan menggunakan angka cost palsu

CONTOH HASIL UI:

LOGS:

Provider: tokenharbor
Model: deepseek-v4-flash:free
Prompt: 120.895
Completion: 220
Total: 121.115
Cost: $0.001

DASHBOARD:

Prompt Tokens: 20.617.868
Completion Tokens: 106.652
Total Tokens: 20.724.520
Estimated Cost (Total): $0.001

Jika terdapat banyak request, contoh:

Request #1 → $0.001
Request #2 → $0.002
Request #3 → $0.001

Dashboard:
Estimated Cost (Total) = $0.004

==================================================
1. COST HARUS PER REQUEST
==================================================

Setiap Usage Record harus memiliki field cost yang dihitung hanya dari token request tersebut.

Gunakan:

cost =
(prompt_tokens / 1,000,000 × input_price_per_1m)
+
(completion_tokens / 1,000,000 × output_price_per_1m)

Jangan menggunakan:
- cumulative token
- session token
- token dari OpenCode
- token dari request sebelumnya
- running total yang berasal dari upstream session.

Cost harus immutable untuk record request tersebut setelah usage final diketahui.

==================================================
2. PRICING PER PROVIDER + MODEL
==================================================

Buat pricing registry/configuration yang terstruktur berdasarkan:

provider + exact model ID

Contoh konsep:

provider: tokenharbor
model: deepseek-v4-flash:free
inputPricePer1M: ...
outputPricePer1M: ...

provider: nvidia
model: exact-model-id
inputPricePer1M: ...
outputPricePer1M: ...

Jangan menganggap semua provider/model mempunyai harga yang sama.

Jangan hardcode satu harga global untuk seluruh provider.

Gunakan exact model ID yang tercatat pada Usage Record.

==================================================
3. JIKA HARGA BELUM TERSEDIA
==================================================

Jika provider/model belum mempunyai pricing:

- jangan mengarang harga
- jangan menganggap harga = $0
- jangan menampilkan cost palsu

Tampilkan:

Cost: N/A

dan dashboard hanya menjumlahkan record yang memang memiliki cost valid.

Namun buat struktur pricing agar harga dapat ditambahkan dengan mudah tanpa mengubah sistem Usage Tracking.

==================================================
4. SIMPAN COST DI USAGE RECORD
==================================================

Tambahkan field:

costUsd

atau nama field yang konsisten dengan schema existing.

Contoh:

{
  promptTokens: 120895,
  completionTokens: 220,
  totalTokens: 121115,
  costUsd: 0.001
}

Pastikan cost berasal dari token request tersebut.

Jangan menyimpan cost berdasarkan cumulative/session usage.

==================================================
5. LOGS
==================================================

Tambahkan kolom:

COST

Contoh:

PROMPT | COMPLETION | TOTAL | COST

120.895 | 220 | 121.115 | $0.001

Format USD:

- gunakan `$0.001`
- jangan tampilkan `$0`
- jangan membulatkan cost kecil menjadi `$0`
- gunakan precision yang cukup untuk micro-cost.

Jika cost sangat kecil, tetap tampilkan nilai yang bermakna.

Gunakan formatting yang konsisten, misalnya:

$0.001
$0.002
$0.015
$1.234

Jangan kehilangan precision karena pembulatan terlalu awal.

PENTING:
Lakukan perhitungan menggunakan angka penuh terlebih dahulu.
Pembulatan hanya dilakukan saat formatting UI.

==================================================
6. DASHBOARD
==================================================

Card:

EST. COST (TOTAL)

harus dihitung:

SUM(costUsd dari setiap usage record valid)

Bukan:

totalTokens × harga

jika `totalTokens` tersebut berasal dari agregasi yang berpotensi cumulative.

Dashboard harus menjumlahkan cost masing-masing record.

Contoh:

record 1 = $0.001
record 2 = $0.002
record 3 = $0.001

Dashboard = $0.004

==================================================
7. PROVIDER USAGE
==================================================

Pada Provider Usage tambahkan:

Cost (USD)

Contoh:

Provider | Requests | Tokens | Cost

tokenharbor | 100 | 1,234,567 | $0.123

nvidia | 50 | 500,000 | $0.050

Cost provider = SUM(costUsd dari request provider tersebut).

==================================================
8. MODEL USAGE
==================================================

Pada Model Usage tambahkan:

Cost (USD)

Cost dihitung dari request individual model tersebut.

Jangan mencampur model berbeda.

==================================================
9. TOKEN NULL
==================================================

Jika usage provider:

promptTokens = null
completionTokens = null
totalTokens = null

maka:

costUsd = null

Jangan mengestimasi cost.

Jika hanya sebagian token tersedia dan pricing tidak memungkinkan perhitungan yang valid:

costUsd = null

==================================================
10. STREAMING
==================================================

Pertahankan behavior streaming yang sudah benar.

Jika final streaming usage tersedia:
→ hitung cost dari usage final tersebut.

Jika streaming usage tetap null:
→ costUsd = null

Jangan mengestimasi token atau cost.

Jangan membuat logging memutus stream.

==================================================
11. OPENAI/OPENCODE CUMULATIVE TOKEN BUG
==================================================

Ini SANGAT PENTING.

Pastikan cost tidak ikut membesar karena hasil cumulative dari OpenCode/session.

Setiap request harus diproses secara independen.

Contoh jika upstream memberikan:

Request 1:
prompt = 100
completion = 20

Request 2:
prompt = 150
completion = 30

Maka:

Request 1 cost = berdasarkan 100 + 20
Request 2 cost = berdasarkan 150 + 30

BUKAN:

Request 2 menggunakan 250 + 50.

Jangan membaca total cumulative dari session sebagai usage request individual.

==================================================
12. BACKWARD COMPATIBILITY
==================================================

Usage record lama yang belum mempunyai `costUsd`:

- jangan rusak
- jangan mengarang cost historical
- boleh tampil `N/A`

Cost baru dihitung mulai dari request yang sudah mempunyai usage valid setelah fitur ini aktif.

==================================================
13. TESTING
==================================================

Tambahkan test:

1. cost request individual benar
2. input pricing benar
3. output pricing benar
4. total cost benar
5. multiple request tidak cumulative
6. dashboard SUM cost benar
7. provider SUM cost benar
8. model SUM cost benar
9. null token menghasilkan null cost
10. streaming usage menghasilkan cost jika tersedia
11. streaming usage null menghasilkan null cost
12. cost tidak berasal dari OpenCode cumulative token
13. cost precision tidak berubah karena formatting UI
14. record lama tanpa cost tetap dapat dibaca.

Gunakan contoh test:

Request A:
prompt = 1000
completion = 500

Request B:
prompt = 2000
completion = 1000

Pastikan B dihitung berdasarkan:
2000 + 1000

dan B TIDAK dihitung berdasarkan:
3000 + 1500.

==================================================
14. UI FORMAT
==================================================

Dashboard:

EST. COST (TOTAL)
$0.001

Logs:

COST
$0.001

Jangan tampilkan:

$0

untuk cost yang sebenarnya non-zero.

Jika cost benar-benar 0 karena pricing gratis:
$0.000

Jika pricing tidak tersedia:
N/A

==================================================
15. VALIDASI AKHIR
==================================================

Setelah implementasi:

- npm run lint
- npm run build
- npm test

Jalankan test usage/cost yang relevan.

Audit source code untuk memastikan tidak ada lagi cost calculation yang membaca cumulative/session token dari OpenCode.

Pastikan:

Usage Record
→ token request individual
→ pricing provider/model
→ costUsd request individual
→ Logs menampilkan cost
→ Provider Usage menjumlahkan cost
→ Model Usage menjumlahkan cost
→ Dashboard menjumlahkan cost.

Jangan merusak:
- Provider Management
- Model Registry
- Usage Tracking
- Usage Logs
- Dashboard
- Enable/Disable Provider
- Streaming
- Backup/Restore
- API endpoint existing.

Jangan membuat mock provider.
Jangan mengarang harga.
Jangan mengarang token.
Jangan menggunakan cumulative token dari OpenCode.
Jangan melakukan refactor besar yang tidak diperlukan.

SETELAH SELESAI:
Laporkan:
- pricing registry yang digunakan
- contoh cost per request
- contoh cost di Logs
- hasil Dashboard Estimated Cost
- hasil Provider Usage
- hasil Model Usage
- bagaimana cumulative OpenCode dicegah
- test pass/fail
- lint
- build
- file yang diubah
- masalah yang masih tersisa.
```
# 
```


Perbaiki integrasi live data Usage & Logs pada `nvidia-api`.

Masalah:
Dashboard UI sudah tampil, tetapi data Usage/Logs belum live dan belum otomatis ter-update setelah request API baru.

Tugas:
- Audit sumber data `/admin/usage`, `/admin/usage/providers`, `/admin/usage/models`, `/admin/usage/records`, dan `/admin/logs`.
- Pastikan UI mengambil data langsung dari backend, bukan data statis/cache lama.
- Setelah request API baru, data Usage dan Logs harus mencerminkan record terbaru.
- Tombol Refresh harus mengambil data terbaru dari backend.
- Jika sudah ada auto-refresh/polling, pastikan benar-benar bekerja.
- Jangan membuat data dummy.
- Jangan mengubah struktur/provider/API yang sudah bekerja.
- Jangan menghapus Usage/Logs existing.
- Pastikan provider, model, status, HTTP status, token, latency, dan timestamp berasal dari record aktual.
- Test dengan membuat 1 request API nyata, lalu refresh dashboard dan pastikan angka/log bertambah sesuai record tersebut.
- Test juga tab Overview, Provider Usage, Model Usage, dan Logs.
- Pastikan tidak ada console error terkait data/API.

Setelah selesai tampilkan:
1. Penyebab data tidak live.
2. File yang diperbaiki.
3. Endpoint yang digunakan UI.
4. Hasil test request → log → dashboard.
5. Pastikan data terbaru muncul tanpa restart server.
6. URL Admin UI.
```

# 
```
Lakukan redesign UI Admin Dashboard pada project `nvidia-api` dengan MENGIKUTI GAMBAR REFERENSI yang sudah disimpan di:

`docs/design/admin-dashboard-reference.png`

TUJUAN:
Buat UI NVIDIA API Admin Dashboard mengikuti desain pada gambar referensi tersebut secara visual dan konsisten.

ATURAN UTAMA:
- Gunakan gambar `docs/design/admin-dashboard-reference.png` sebagai sumber utama desain.
- Jangan membuat gambar/desain baru.
- Jangan hanya meniru warna; ikuti struktur/layout, spacing, ukuran card, typography, navigation, button, table, status badge, dan responsive behavior.
- Gambar referensi memiliki tampilan desktop dan mobile. Gunakan bagian masing-masing sebagai acuan responsive.
- UI harus tetap menggunakan data/API/backend yang sudah ada.
- Jangan mengubah backend, provider registry, usage tracking, backup, API endpoint, atau logic bisnis.
- Jangan menambah fitur baru.
- Fokus hanya pada UI/UX dan styling.

YANG HARUS DISESUAIKAN:
1. Overall layout
   - Header/top bar
   - Sidebar/navigation atau tab navigation
   - Content container
   - Section spacing
   - Card layout
   - Border radius
   - Border/shadow
   - Typography hierarchy

2. Dashboard
   - Usage Summary
   - Total Requests
   - Successful
   - Errors
   - Blocked
   - Prompt Tokens
   - Completion Tokens
   - Total Tokens
   - Average Latency
   - Registered Providers

3. Navigation
   Pertahankan section yang sudah ada:
   - Overview
   - Providers
   - Provider Usage
   - Model Usage
   - Logs
   - Backup

   Hanya ubah tampilan agar mengikuti referensi.

4. Provider UI
   - Provider list
   - Provider name
   - Provider ID
   - jumlah model
   - status Enabled/Disabled
   - tombol Enable/Disable

   Jangan mengubah logic enable/disable.

5. Usage / Logs / Backup
   Semua halaman harus menggunakan design language yang sama dengan reference.
   Jangan mengubah fungsi existing.

6. RESPONSIVE
   WAJIB cek:
   - Desktop
   - Tablet jika relevan
   - Mobile

   Mobile harus mengikuti bagian mobile pada reference.
   Jangan sampai:
   - horizontal overflow
   - tabel keluar layar
   - card terpotong
   - button keluar container
   - text overlap
   - navigation rusak.

7. VISUAL QUALITY
   Pastikan hasil akhir terlihat seperti satu produk yang konsisten:
   - spacing konsisten
   - font hierarchy jelas
   - card tidak terlalu besar
   - informasi mudah dipindai
   - status badge jelas
   - button konsisten
   - warna mengikuti reference
   - jangan menambahkan dekorasi yang tidak ada di reference.

8. IMPLEMENTATION
   Sebelum mengubah code:
   - audit UI existing
   - cari file frontend/admin yang digunakan
   - identifikasi component/style yang perlu diubah
   - jangan membuat duplicate UI system jika component existing masih bisa digunakan.

   Setelah perubahan:
   - jalankan build
   - jalankan test UI yang tersedia
   - jalankan lint/typecheck jika tersedia.

9. UI TEST
   Jalankan server NVIDIA API.
   Gunakan browser/Playwright/Puppeteer yang tersedia untuk membuka:

   `http://127.0.0.1:3000/admin`

   Test minimal:
   - Overview
   - Providers
   - Provider Usage
   - Model Usage
   - Logs
   - Backup
   - desktop viewport
   - mobile viewport

   Periksa:
   - tidak ada console error selain error benign yang sudah diketahui
   - tidak ada horizontal overflow
   - semua navigation dapat diklik
   - data tetap muncul
   - Enable/Disable tetap bekerja
   - Backup UI tetap bekerja
   - tabel/card responsive.

10. JANGAN MENGUBAH LOGIC
   Jangan mengubah:
   - provider registry
   - provider discovery
   - model registry
   - API request
   - usage tracking
   - logs backend
   - backup/restore backend
   - authentication
   - credential
   - endpoint API

HASIL AKHIR:
Laporkan:
- file UI yang diubah
- component/style yang diubah
- desktop test result
- mobile test result
- halaman yang sudah mengikuti reference
- console error jika ada
- overflow/responsive issue jika ada
- build result
- test result

PENTING:
Ini hanya REDESIGN UI berdasarkan reference.
Jangan menambah fitur.
Jangan membuat gambar baru.
Jangan mengubah backend.
Jangan mengubah behavior/API existing.


```


# Prompt 12 — Final Audit Existing Features
```

Lanjutkan project `nvidia-api`.

PENTING:
JANGAN MENAMBAHKAN FITUR BARU.

Pada tahap ini jangan membuat API Key Management, Quota, Client Management, Add Provider UI, OAuth, billing, atau fitur baru lainnya.

SCOPE FINAL PROJECT SAAT INI HANYA:
1. Usage Tracking
2. Usage Dashboard
3. Usage Logs
4. Provider Enable/Disable
5. Backup & Restore
6. Multi-provider melalui provider registry/code yang sudah ada

Provider baru MASIH ditambahkan melalui CODE/provider registry.
JANGAN membuat UI atau endpoint "Add Provider".
JANGAN mengubah arsitektur provider menjadi dynamic database provider management.

Provider yang sudah ada harus tetap dapat bekerja melalui registry/configuration yang sekarang.

==================================================
1. AUDIT KONDISI SEKARANG
==================================================

Audit seluruh implementasi existing setelah Prompt 1–11.

Periksa:

- Provider Registry
- Model Registry
- Provider Enable/Disable
- Usage Tracking
- Usage Dashboard
- Usage Logs
- Backup
- Restore
- `/v1/models`
- endpoint API existing
- persistence
- security
- streaming
- provider discovery

Jangan melakukan refactor besar.

Jika implementasi sudah benar:
JANGAN ubah hanya untuk mempercantik kode.

Jika ditemukan bug:
perbaiki root cause sekecil mungkin.

==================================================
2. PROVIDER ARCHITECTURE
==================================================

Pastikan project tetap MULTI-PROVIDER.

Provider berasal dari source code/provider registry yang sudah ada.

Contoh provider yang saat ini relevan:
- NVIDIA
- TokenHarbor.ai
- provider lain yang memang sudah terdaftar di code

Jangan hardcode hanya NVIDIA.

Namun JANGAN membuat fitur Add Provider.

Provider baru tetap dilakukan dengan:
- menambahkan provider implementation
- mendaftarkan provider pada registry
- menambahkan konfigurasi credential sesuai arsitektur existing.

Admin UI hanya boleh:
- melihat provider
- melihat model
- enable provider
- disable provider.

Tidak boleh:
- Add Provider
- Edit Provider
- Delete Provider
- memasukkan provider baru melalui UI.

==================================================
3. PROVIDER ENABLE/DISABLE
==================================================

Pastikan setiap provider yang terdaftar melalui code dapat:

ACTIVE
→ menerima request

DISABLED
→ request baru diblokir

ENABLE kembali
→ request dapat digunakan kembali.

State harus persistent.

Restart server tidak boleh menghilangkan state disable.

Provider disabled:
- tidak boleh menerima request upstream
- tidak boleh fallback diam-diam ke provider lain
- harus tercatat sebagai blocked sesuai Usage schema existing.

Jangan menghapus provider/model dari registry hanya karena disabled.

==================================================
4. MODEL REGISTRY
==================================================

Model harus berasal dari provider registry/discovery yang benar-benar digunakan.

Jangan membuat model dummy.

Jangan hardcode model hanya agar test lulus.

`/v1/models` harus menampilkan model yang benar-benar tersedia sesuai behavior existing.

Provider disabled boleh tetap dikenal oleh admin/registry sesuai desain existing, tetapi model tersebut tidak boleh menerima request ketika provider disabled.

==================================================
5. USAGE TRACKING
==================================================

Pertahankan Usage Tracking yang sudah ada.

Setiap request yang relevan harus dapat mencatat:

- timestamp
- provider
- exact model
- status
- HTTP status
- prompt/input tokens
- completion/output tokens
- total tokens
- latency
- error message/code jika ada
- request ID jika tersedia
- client/API identifier yang memang sudah ada pada sistem

JANGAN membuat sistem usage kedua.

JANGAN membuat database usage baru.

Gunakan storage existing.

Token harus berasal dari upstream.

Jika provider tidak mengirim usage:
- tetap null sesuai schema existing
- jangan estimasi
- jangan mengarang token.

Pastikan:

prompt_tokens + completion_tokens = total_tokens

jika semua nilai tersedia.

==================================================
6. USAGE STATUS
==================================================

Pastikan Usage dapat membedakan:

SUCCESS
BLOCKED
ERROR

Contoh:

HTTP 200
→ success

Provider disabled
→ blocked

Invalid model
→ blocked/validation sesuai behavior existing

Upstream 401/403/429/5xx
→ error

Jangan mengubah status sebenarnya hanya agar dashboard terlihat bagus.

==================================================
7. HTTP STATUS
==================================================

Pastikan masalah Prompt 8 tetap terselesaikan.

Success request:
httpStatus = 200

Upstream error:
httpStatus = status asli upstream

Blocked:
gunakan status existing yang memang digunakan project.

Jangan mengarang HTTP status.

Pastikan nilai tersebut konsisten pada:

- Usage Logs
- `/admin/usage`
- `/admin/usage/providers`
- `/admin/usage/models`
- `/admin/usage/records`
- `/admin/logs`

==================================================
8. STREAMING
==================================================

Audit streaming existing.

Pastikan:

- stream dimulai dengan benar
- stream tidak diputus oleh Usage Logging
- stream selesai normal
- Usage Log tetap dibuat.

Jika final chunk upstream memiliki:

usage: null

maka:
- simpan token sebagai null
- jangan estimasi
- jangan mengubah menjadi error.

Jika upstream memberikan usage:
- simpan usage asli.

Jangan melakukan tokenizer tambahan hanya untuk menghasilkan angka token.

==================================================
9. USAGE DASHBOARD
==================================================

Jangan menambahkan analytics baru.

Pastikan dashboard existing menampilkan dengan benar:

TOTAL
- requests
- success
- error
- blocked
- input tokens
- output tokens
- total tokens
- average latency jika memang sudah ada.

PER PROVIDER
- provider
- requests
- success/error/blocked
- input tokens
- output tokens
- total tokens
- latency jika tersedia.

PER MODEL
- exact model
- provider
- requests
- tokens
- status.

Jangan membuat fitur billing/quota.

==================================================
10. LOGS
==================================================

Pastikan Logs existing dapat:

- menampilkan record
- filter provider
- filter model
- filter status
- pagination
- melihat detail jika sudah ada.

Jangan menampilkan:

- raw API key
- provider secret
- Authorization header
- credential.

Provider dan model harus berasal dari request sebenarnya.

==================================================
11. BACKUP
==================================================

Pertahankan sistem Backup/Restore yang sudah dibuat.

Backup minimal harus mempertahankan:

- Usage records
- Provider state enabled/disabled
- persistent state lain yang memang diperlukan existing
- metadata backup
- backupVersion.

Jangan memasukkan:

- NVIDIA API key
- TokenHarbor API key
- provider credentials
- Authorization header
- `.env`
- private key
- password
- secret.

Jangan menambahkan encryption baru jika project belum memiliki mekanisme encryption yang benar.

Jangan membuat encryption palsu.

==================================================
12. RESTORE
==================================================

Audit restore existing.

Restore harus:

1. validasi backup
2. validasi version
3. validasi struktur
4. membuat pre-restore backup
5. restore dataset
6. restore provider state
7. memastikan data dapat dibaca kembali.

Tidak boleh ada automatic restore saat startup.

Jika restore gagal:
- jangan meninggalkan state setengah restore jika dapat dihindari
- tampilkan error yang jelas.

==================================================
13. BACKUP RETENTION
==================================================

Jangan menambah fitur retention baru.

Pertahankan behavior retention yang sudah dibuat sebelumnya.

Jika `BACKUP_MAX_BACKUPS` sudah ada:
- pastikan tetap bekerja sesuai konfigurasi.

Jangan menghapus backup lama tanpa konfigurasi.

==================================================
14. SECURITY AUDIT
==================================================

Scan source, logs, usage records, dan backup.

Pastikan:

0 raw NVIDIA API key
0 raw TokenHarbor API key
0 Authorization header
0 provider credential
0 `.env` secret
0 private key.

Masked value boleh jika memang diperlukan.

Jangan menampilkan secret dalam laporan.

==================================================
15. REAL PROVIDER TEST
==================================================

Lakukan verification hanya untuk provider nyata yang memang ditentukan:

NVIDIA
TokenHarbor.ai

WAJIB:
- NVIDIA boleh dan harus dites jika credential tersedia.
- TokenHarbor.ai boleh dan harus dites jika credential tersedia.

DILARANG:
- Gorouter.app
- credential Gorouter
- Gorouter sebagai fallback
- Gorouter sebagai model discovery source
- live Gorouter integration test.

Jangan membuat provider dummy.

==================================================
16. NVIDIA TEST
==================================================

Gunakan model NVIDIA yang benar-benar ditemukan.

Prioritas:

`deepseek-ai/deepseek-v4-flash-0731`

Jika tersedia dan credential memiliki inference permission:

- request nyata
- HTTP status
- response
- provider
- exact model
- token usage
- latency
- Usage Log.

Test provider:

enabled
→ request

disabled
→ blocked

enabled kembali
→ request.

Jika credential NVIDIA mendapat 401/403:
- jangan bypass
- laporkan status sebenarnya.

==================================================
17. TOKENHARBOR TEST
==================================================

Gunakan model TokenHarbor yang benar-benar tersedia.

Jika credential tersedia:

- request nyata
- exact provider ID
- exact model ID
- HTTP status
- usage jika tersedia
- latency
- Usage Log.

Test:

enabled
→ request

disabled
→ blocked

enabled kembali
→ request.

Jika TokenHarbor credential tidak memiliki inference permission:
- jangan bypass
- laporkan status sebenarnya.

==================================================
18. BACKUP REAL DATA
==================================================

Setelah real provider test jika credential tersedia:

1. Buat usage records.
2. Buat backup.
3. Periksa backup.
4. Pastikan usage records masuk.
5. Pastikan provider state masuk.
6. Pastikan credential TIDAK masuk.
7. Restore.
8. Pastikan usage dapat dibaca.
9. Pastikan provider state tetap benar.

Jangan menghapus production data.

==================================================
19. TEST SUITE
==================================================

Jalankan:

npm run lint
npm run build

Untuk test:

JANGAN menjalankan test/integration test Gorouter.app.

NVIDIA dan TokenHarbor.ai boleh dites.

Jika `npm test` otomatis menjalankan test Gorouter:
- skip/exclude test Gorouter.
- jangan memodifikasi test hanya agar pass.
- laporkan skipped test.

`models.test.ts` yang membutuhkan:

GROUTER_API_KEY
atau
GROUTER_API_KEYS

tetap dianggap pre-existing/environment limitation jika memang bukan akibat perubahan saat ini.

Jangan membuat credential Gorouter.

==================================================
20. REGRESSION
==================================================

Pastikan tidak merusak:

- provider registry
- model registry
- provider discovery
- provider enable/disable
- `/v1/models`
- normal API request
- streaming
- Usage Tracking
- Usage Dashboard
- Usage Logs
- Backup
- Restore
- persistence.

Jangan menambahkan fitur lain.

==================================================
21. DOKUMENTASI
==================================================

Update dokumentasi existing hanya jika diperlukan.

Dokumentasikan bahwa:

- provider baru masih ditambahkan melalui code/provider registry.
- admin hanya dapat enable/disable provider.
- Usage dan Backup/Restore adalah fitur utama.
- provider credential tidak masuk backup.
- Gorouter tidak digunakan dalam production verification.

Jangan membuat banyak README baru.

==================================================
22. HASIL AKHIR
==================================================

Berikan laporan lengkap:

1. Status Provider Registry
2. Provider yang terdaftar
3. Model yang ditemukan
4. Status Enable/Disable
5. Usage Tracking
6. Usage Dashboard
7. Usage Logs
8. Backup
9. Restore
10. Security Audit
11. NVIDIA test
12. TokenHarbor.ai test
13. Exact model ID yang dites
14. HTTP status
15. Token usage
16. Latency
17. Streaming
18. npm run lint
19. npm run build
20. npm test
21. jumlah pass/fail/skip
22. status `models.test.ts`
23. file yang berubah
24. masalah yang masih tersisa.

PENTING TERAKHIR:

JANGAN MENAMBAHKAN FITUR BARU.

Jangan membuat:
- API Key Management
- Client Management
- Quota
- Billing
- Add Provider UI
- Delete Provider UI
- OAuth
- Dynamic Provider CRUD
- provider database baru
- database usage baru.

Fokus hanya menyelesaikan dan memastikan fitur existing:

USAGE
+
ENABLE/DISABLE PROVIDER
+
BACKUP/RESTORE

Provider baru tetap melalui CODE/provider registry.

NVIDIA dan TokenHarbor.ai adalah provider untuk verification.

GOROUTER.APP SAMA SEKALI JANGAN DIGUNAKAN.

```


# 
```

PROMPT 13 — FINAL ADMIN DASHBOARD INTEGRATION, PROVIDER/MODEL MANAGEMENT, USAGE, BACKUP & RESTORE

Gunakan model GLM 5.2 untuk mengerjakan prompt ini.

Lanjutkan project `nvidia-api` dari kondisi saat ini. Jangan mengulang, merusak, atau mengganti fitur yang sudah selesai pada Prompt 1–12.

TUJUAN:
Selesaikan integrasi Admin Dashboard agar seluruh fitur backend yang sudah dibuat benar-benar tersedia dan nyaman digunakan dari UI admin.

FITUR YANG WAJIB TERINTEGRASI:
- Provider Management
- Enable/Disable Provider
- Model Registry
- Model per Provider
- Usage Dashboard
- Usage per Provider
- Usage per Model
- Total Token
- Usage Logs
- Log Detail
- Filter
- Pagination
- Backup
- Restore
- Security Masking

==================================================
1. AUDIT UI EXISTING
==================================================

Sebelum mengubah kode:

- audit struktur frontend/admin yang sudah ada
- cari route/page admin existing
- cari komponen dashboard existing
- cari API client/service existing
- gunakan struktur UI existing
- jangan membuat dashboard baru yang duplikatif
- jangan mengganti framework frontend
- jangan melakukan refactor besar tanpa alasan

Jika sudah ada halaman/section yang sesuai, integrasikan fitur ke halaman tersebut.

==================================================
2. PROVIDER MANAGEMENT
==================================================

Buat/sempurnakan halaman Providers.

Tampilkan provider yang benar-benar terdaftar pada runtime.

Setiap provider minimal:

- Provider name
- Provider ID
- Status Active/Disabled
- jumlah model
- daftar model atau tombol melihat model
- Enable
- Disable

Provider harus berasal dari backend registry/runtime.

Jangan menampilkan provider dummy.

Provider utama untuk real testing:

- NVIDIA
- TokenHarbor.ai

Gorouter.app JANGAN digunakan.

==================================================
3. ENABLE / DISABLE PROVIDER
==================================================

Tombol Enable/Disable harus memanggil API backend existing.

Flow:

Active
→ Disable
→ confirmation
→ API backend
→ refresh state
→ Disabled

Disabled
→ Enable
→ API backend
→ refresh state
→ Active

State harus benar setelah:

- refresh browser
- server restart

Jangan hanya mengubah state frontend.

Backend tetap menjadi source of truth.

==================================================
4. MODEL PER PROVIDER
==================================================

Tampilkan model yang benar-benar tersedia dari provider registry/discovery.

Minimal:

Provider
├── Status
├── Model count
└── Models
    ├── Exact Model ID
    ├── Provider
    └── Status

Jangan hardcode model yang tidak ditemukan dari backend.

Jika DeepSeek V4 Pro atau GLM 5.2 tersedia dari provider yang benar-benar terdaftar, gunakan exact model ID hasil discovery.

Jangan membuat model palsu.

==================================================
5. MODEL STATUS
==================================================

Jika backend sudah mendukung model enable/disable:

tampilkan:

- model ID
- provider
- Enabled/Disabled

Jika backend belum mendukung model enable/disable secara resmi:

JANGAN membuat sistem backend baru hanya untuk UI.

Cukup tampilkan status yang tersedia.

Provider disabled tetap harus memblokir request model provider tersebut.

==================================================
6. USAGE DASHBOARD
==================================================

Gunakan endpoint Usage existing.

Summary:

- Total Requests
- Successful
- Errors
- Blocked
- Prompt Tokens
- Completion Tokens
- Total Tokens
- Average Latency

Jangan menghitung token dari frontend.

Backend adalah source of truth.

Jika token null:

tampilkan `—` atau null sesuai UI.

Jangan melakukan estimasi token.

==================================================
7. USAGE PER PROVIDER
==================================================

Buat tabel:

Provider
Requests
Success
Error
Blocked
Prompt Tokens
Completion Tokens
Total Tokens
Average Latency

Provider harus berasal dari Usage API.

Jangan memasukkan provider dummy.

==================================================
8. USAGE PER MODEL
==================================================

Buat tabel:

Model
Provider
Requests
Success
Error
Blocked
Prompt Tokens
Completion Tokens
Total Tokens
Average Latency

Gunakan exact model ID dari backend.

Jangan mempersingkat atau mengubah model ID secara internal.

==================================================
9. USAGE LOGS
==================================================

Buat/sempurnakan halaman Logs.

Minimal:

- Timestamp
- Provider
- Model
- Status
- HTTP Status
- Prompt Tokens
- Completion Tokens
- Total Tokens
- Latency
- Request ID
- Client/API identifier masked

Status harus dibedakan:

- success
- error
- blocked

Gunakan status backend.

==================================================
10. LOG DETAIL
==================================================

Saat admin membuka satu log:

Tampilkan:

- Timestamp
- Provider
- Model
- Status
- HTTP Status
- Latency
- Prompt Tokens
- Completion Tokens
- Total Tokens
- Request ID
- Client ID
- Error Message

JANGAN tampilkan:

- API key asli
- Authorization header
- NVIDIA API key
- TokenHarbor API key
- provider secret
- .env
- credential lainnya

Jika backend memberikan apiKeyMasked, gunakan nilai tersebut.

==================================================
11. FILTER LOGS
==================================================

Tambahkan:

- Provider
- Model
- Status
- From
- To
- Search Request ID

Gunakan filter backend existing.

Jangan mengambil seluruh dataset lalu filtering hanya di frontend jika backend sudah menyediakan filter.

==================================================
12. PAGINATION
==================================================

Logs wajib menggunakan pagination.

Gunakan mekanisme existing seperti:

limit
offset

UI:

Previous
1 2 3 ...
Next

Jangan mengambil seluruh log sekaligus.

==================================================
13. REFRESH
==================================================

Tambahkan tombol Refresh.

Refresh harus mengambil data terbaru dari backend.

Jangan menggunakan polling agresif.

Tidak perlu websocket.

Loading:

Loading...

Error:

Failed to load data
Retry

Empty:

No data found

Pastikan array kosong tidak menyebabkan crash.

==================================================
14. PROVIDER + USAGE INTEGRATION
==================================================

Ketika provider disabled:

Provider
→ Disabled
→ request baru
→ blocked
→ Usage Log = blocked

Saat enabled:

Provider
→ Enabled
→ request dapat diproses

Jangan hanya mengubah tampilan frontend.

==================================================
15. NVIDIA REAL VERIFICATION
==================================================

NVIDIA adalah provider real.

Jika credential NVIDIA valid, lakukan real test.

Prioritas model:

deepseek-ai/deepseek-v4-flash-0731

Jika tersedia dan memiliki inference access:

- request nyata
- HTTP 200
- response valid
- provider = nvidia
- exact model ID benar

Boleh menggunakan model NVIDIA lain yang benar-benar tersedia.

Jangan membuat model dummy.

==================================================
16. TOKENHARBOR.AI REAL VERIFICATION
==================================================

TokenHarbor.ai juga WAJIB menjadi provider real.

Gunakan credential TokenHarbor jika tersedia.

Model harus berasal dari discovery/configuration TokenHarbor yang nyata.

Test:

- request nyata
- provider benar
- exact model benar
- HTTP status
- token usage jika tersedia
- latency
- Usage Log

Jika credential tidak tersedia atau tidak memiliki inference access:

JANGAN membuat credential dummy.

Laporkan error sebenarnya.

==================================================
17. GOROUTER.APP
==================================================

Gorouter.app TIDAK DIGUNAKAN.

JANGAN:

- menjalankan test Gorouter
- melakukan integration test Gorouter
- menggunakan Gorouter sebagai fallback
- menggunakan Gorouter sebagai proxy
- menggunakan credential Gorouter
- menggunakan model Gorouter
- memasukkan Gorouter ke provider runtime

Jika test suite otomatis menemukan test Gorouter:

- skip/exclude
- jangan mengubah test untuk memalsukan hasil
- laporkan test yang di-skip

Fokus provider real:

NVIDIA
TokenHarbor.ai

==================================================
18. INVALID MODEL
==================================================

Gunakan model ID yang benar-benar tidak tersedia.

Pastikan:

- request ditolak
- tidak diteruskan ke provider
- tidak fallback ke Gorouter
- tidak tercatat sebagai success
- log mencatat error/blocked sesuai behavior existing

==================================================
19. UPSTREAM ERROR
==================================================

Jika dapat diuji secara aman:

- upstream error harus dicatat sebagai error
- HTTP status aktual disimpan
- error message disimpan
- token tetap null jika upstream tidak memberikan usage

Jangan membuat error palsu.

==================================================
20. STREAMING
==================================================

Jika provider mendukung streaming:

NVIDIA:
- test streaming jika credential valid

TokenHarbor:
- test streaming jika tersedia dan credential valid

Pastikan:

- stream selesai normal
- Usage Log dibuat
- usage disimpan jika tersedia
- usage null tetap null jika upstream tidak memberikan
- jangan mengestimasi token
- logging tidak memutus stream

==================================================
21. ADMIN USAGE ENDPOINT
==================================================

Verifikasi endpoint existing:

GET /admin/usage
GET /admin/usage/providers
GET /admin/usage/models
GET /admin/usage/records
GET /admin/logs

Pastikan data konsisten dengan request nyata.

==================================================
22. BACKUP UI
==================================================

Integrasikan fitur Backup/Restore existing.

Gunakan endpoint:

GET /admin/backup/list
GET /admin/backup/info/:id
GET /admin/backup/download/:id
POST /admin/backup/restore/:id
DELETE /admin/backup/:id

Tampilkan:

- Backup ID
- Created At
- Size
- Usage Record Count
- Version
- Valid/Invalid

Action:

Create Backup
Download
Info
Restore
Delete

Jangan membuat endpoint backup duplikatif.

==================================================
23. CREATE BACKUP
==================================================

Saat admin menekan Create Backup:

- loading
- panggil backend
- tampilkan success/error
- refresh daftar backup

Backup harus mencakup data persistent penting seperti:

- Usage records
- Usage Logs
- Provider state
- persistent state yang memang diperlukan

Jangan memasukkan secret.

==================================================
24. RESTORE
==================================================

Restore adalah operasi sensitif.

Sebelum restore tampilkan confirmation:

Backup ID
Timestamp
Usage Record Count
Version

Tombol:

Restore
Cancel

Jangan melakukan restore hanya karena halaman dibuka.

Jika backend membuat pre-restore backup:

tampilkan hasilnya.

==================================================
25. BACKUP SECURITY
==================================================

Backup TIDAK BOLEH berisi:

- NVIDIA API key
- TokenHarbor API key
- Authorization header
- client secret
- provider credential
- .env
- password
- private key

Gunakan filtering existing.

Jangan membuat encryption palsu.

==================================================
26. ERROR HANDLING
==================================================

Semua API call harus menangani:

- loading
- success
- empty
- error

Pastikan tidak crash ketika:

tokens = null
latency = null
httpStatus = null
errorMessage = null

==================================================
27. RESPONSIVE UI
==================================================

Admin UI harus usable pada:

- desktop
- tablet
- mobile

Untuk tabel besar boleh menggunakan horizontal scroll.

Jangan membuat layout mobile rusak.

==================================================
28. PERFORMANCE
==================================================

Jangan:

- mengambil seluruh logs
- polling agresif
- request API berulang tanpa alasan
- agregasi besar di frontend
- duplicate request saat render

Gunakan endpoint aggregation/pagination existing.

==================================================
29. FRONTEND TEST
==================================================

Jika project memiliki frontend testing:

tambahkan test untuk:

- provider list
- enable provider
- disable provider
- provider refresh
- model list
- usage summary
- provider usage
- model usage
- logs
- filter
- pagination
- log detail
- backup list
- create backup
- restore confirmation
- empty state
- error state
- null token handling
- credential masking

==================================================
30. BACKEND REGRESSION
==================================================

Jangan merusak:

- Provider Registry
- Provider Enable/Disable
- Model Registry
- /v1/models
- Usage Tracking
- Usage Dashboard API
- Usage Logs
- Backup
- Restore
- Streaming
- NVIDIA
- TokenHarbor.ai

Jangan melakukan refactor besar.

==================================================
31. TEST SUITE
==================================================

Jalankan:

npm run lint
npm run build

Untuk test internal:

npm test

Tetapi:

JANGAN menjalankan test/integration test Gorouter.app.

Jika test suite otomatis menjalankan Gorouter:

- skip/exclude test tersebut
- jangan memalsukan hasil
- jangan mengubah production code hanya agar Gorouter test pass

NVIDIA dan TokenHarbor.ai boleh dan harus dites jika credential valid.

==================================================
32. SECURITY AUDIT
==================================================

Scan source, logs, backup, dan frontend.

Pastikan tidak ada:

nvapi-...
Authorization: Bearer ...
NVIDIA_API_KEY
TOKENHARBOR_API_KEY
raw API key
.env
private key
password
provider secret

Credential tidak boleh masuk:

- frontend bundle
- logs
- backup
- API response
- UI

==================================================
33. END-TO-END AUDIT
==================================================

Verifikasi alur:

Admin UI
↓
Admin API
↓
Provider Registry
↓
Model Registry
↓
Provider
↓
Upstream
↓
Usage Tracking
↓
Usage Logs
↓
Usage Dashboard
↓
Backup

Ambil minimal satu request nyata jika credential tersedia.

Contoh data:

Provider: nvidia
Model: exact model ID
Status: success
HTTP: 200
Prompt Tokens: actual
Completion Tokens: actual
Total Tokens: actual
Latency: actual

Data tersebut harus konsisten pada:

- API response
- Usage record
- Provider breakdown
- Model breakdown
- Logs
- Log detail
- Backup

==================================================
34. FINAL TEST MATRIX
==================================================

NVIDIA:

Enabled
→ real request
→ success jika credential valid

Disabled
→ request
→ blocked

Enabled kembali
→ request
→ success jika credential valid

TokenHarbor.ai:

Enabled
→ real request
→ success jika credential valid

Disabled
→ request
→ blocked

Enabled kembali
→ request
→ success jika credential valid

Invalid model:

→ blocked/validation error
→ tidak fallback

Upstream error:

→ status error
→ HTTP status aktual

Streaming:

→ stream normal
→ usage actual atau null

Backup:

→ create
→ list
→ info
→ download
→ restore
→ verify

==================================================
35. HASIL AKHIR
==================================================

Setelah selesai berikan laporan lengkap:

UI:
- Provider Management
- Enable/Disable
- Model Registry
- Usage Dashboard
- Provider Usage
- Model Usage
- Logs
- Log Detail
- Filter
- Pagination
- Backup
- Restore

Provider:
- NVIDIA status
- TokenHarbor status
- model yang ditemukan
- exact model ID

Real Test:
- NVIDIA result
- TokenHarbor result
- HTTP status
- token usage
- latency
- streaming

Backup:
- backup result
- restore result
- security result

Testing:
- npm run lint
- npm run build
- npm test
- jumlah pass/fail/skip
- Gorouter test yang di-skip

Security:
- API key masking
- credential protection
- backup security
- frontend secret audit

Regression:
- Provider Management
- Enable/Disable
- Model Registry
- /v1/models
- Usage
- Logs
- Backup
- Restore
- NVIDIA
- TokenHarbor.ai
- Streaming

Jika ada masalah, tuliskan masalah sebenarnya.

JANGAN:

- test Gorouter.app
- menggunakan Gorouter sebagai fallback
- menggunakan credential Gorouter
- membuat provider dummy
- membuat model dummy
- membuat token dummy
- mengestimasi token
- membocorkan API key
- membuat database baru tanpa kebutuhan
- membuat sistem Usage kedua
- membuat sistem Backup kedua
- membuat endpoint duplikat
- melakukan refactor besar yang tidak diperlukan.

Fokus utama project sekarang adalah:

NVIDIA + TokenHarbor.ai

bukan Gorouter.app.

```

# 
```
Lanjutkan project `nvidia-api`.

PROMPT 12 — FULL ADMIN DASHBOARD INTEGRATION

Tujuan utama tahap ini adalah menyelesaikan Admin Dashboard agar seluruh fitur Provider Management, Usage Tracking, Model Registry, Usage Logs, dan statistik token dapat digunakan melalui UI admin dengan data REAL dari backend.

JANGAN membuat data dummy.
JANGAN membuat provider/model dummy.
JANGAN menggunakan Gorouter.app.
JANGAN melakukan live test Gorouter.
NVIDIA dan TokenHarbor.ai tetap dipertahankan sebagai provider yang harus didukung.
Gunakan endpoint/backend existing sebisa mungkin.

==================================================
1. AUDIT STRUKTUR PROJECT SEBELUM MENGUBAH FILE
==================================================

Sebelum coding:

- audit struktur frontend/admin yang sudah ada
- cari routing admin
- cari layout/sidebar/navigation admin
- cari komponen table/card/badge/button existing
- cari API client/service existing
- cari endpoint `/admin/providers`
- cari endpoint `/admin/usage`
- cari endpoint `/admin/usage/providers`
- cari endpoint `/admin/usage/models`
- cari endpoint `/admin/usage/records`
- cari endpoint `/admin/logs`
- cari endpoint model registry
- cari komponen UI yang sudah digunakan project

Jangan membuat sistem frontend kedua.

Gunakan arsitektur dan style existing.

Jangan melakukan refactor besar hanya untuk membuat dashboard.

==================================================
2. ADMIN PROVIDER MANAGEMENT
==================================================

Buat/rapikan halaman Provider Management.

Data harus berasal dari:

GET `/admin/providers`

Tampilkan semua provider yang benar-benar terdaftar.

Untuk setiap provider tampilkan:

- Provider Name
- Provider ID
- Status
- jumlah model
- model yang tersedia
- enabled/disabled state

Status harus jelas:

ENABLED
DISABLED

Gunakan badge/status indicator yang konsisten dengan UI existing.

==================================================
3. TOMBOL ENABLE / DISABLE PROVIDER
==================================================

Tambahkan tombol:

Enable Provider
Disable Provider

Gunakan endpoint existing:

PATCH `/admin/providers/:providerId`

dengan:

{
  "enabled": true
}

atau:

{
  "enabled": false
}

Jangan membuat endpoint baru jika endpoint existing sudah bekerja.

Behavior:

Disable:
- provider menjadi disabled
- provider tidak menerima request baru
- model/provider tetap dikenal oleh registry/admin
- data Usage/Logs lama tidak dihapus
- status persistent
- setelah reload halaman status tetap disabled

Enable:
- provider kembali aktif
- status UI diperbarui
- provider dapat digunakan kembali
- data lama tetap ada

Setelah action berhasil:
- refresh data provider
- jangan hanya mengubah state frontend secara lokal jika backend belum berhasil.

Jika backend mengembalikan error:
- tampilkan error
- jangan menampilkan provider sebagai enabled/disabled secara palsu.

==================================================
4. PROVIDER DETAIL
==================================================

Jika UI existing memungkinkan, buat detail/expand provider.

Tampilkan:

Provider:
- name
- id
- status

Models:
- model ID
- model status jika tersedia

Contoh model NVIDIA/TokenHarbor harus berasal dari backend.

Jangan hardcode:

- NVIDIA models
- TokenHarbor models
- DeepSeek models
- GLM models

Semua harus berasal dari registry/discovery/backend.

==================================================
5. USAGE DASHBOARD
==================================================

Gunakan:

GET `/admin/usage`

Tampilkan summary cards:

- Total Requests
- Successful Requests
- Error Requests
- Blocked Requests
- Prompt/Input Tokens
- Completion/Output Tokens
- Total Tokens
- Average Latency

Jika backend memberikan nilai null:
- jangan mengarang angka
- tampilkan `—` atau `N/A`.

Total token harus berasal dari usage data.

Jangan melakukan token estimation di frontend.

==================================================
6. TOKEN DISPLAY
==================================================

Pastikan UI membedakan:

Prompt/Input Tokens
Completion/Output Tokens
Total Tokens

Contoh:

Prompt:
10

Completion:
6

Total:
16

Jika backend mengatakan:

prompt = 10
completion = 6
total = 16

UI harus menampilkan nilai tersebut secara langsung.

Jangan menghitung ulang token di frontend jika backend sudah memberikan `total_tokens`.

Jika upstream tidak memberikan usage:

Prompt: —
Completion: —
Total: —

Jangan mengubah null menjadi angka palsu.

==================================================
7. USAGE PER PROVIDER
==================================================

Gunakan:

GET `/admin/usage/providers`

Buat tabel/provider breakdown.

Kolom:

- Provider
- Requests
- Success
- Error
- Blocked
- Prompt Tokens
- Completion Tokens
- Total Tokens
- Average Latency

Pastikan provider berasal dari usage record yang sebenarnya.

Contoh:

NVIDIA
TokenHarbor

Jangan menambahkan provider yang tidak dikembalikan backend.

==================================================
8. USAGE PER MODEL
==================================================

Gunakan:

GET `/admin/usage/models`

Tampilkan:

- Model
- Provider
- Requests
- Success
- Error
- Blocked
- Prompt Tokens
- Completion Tokens
- Total Tokens
- Average Latency jika tersedia

Model harus exact model ID dari backend.

Jangan memotong atau mengganti nama model sehingga ID aslinya hilang.

Model seperti:

`deepseek-ai/deepseek-v4-flash-0731`

harus tetap dapat dilihat sebagai exact ID.

==================================================
9. USAGE LOGS
==================================================

Gunakan:

GET `/admin/usage/records`

atau endpoint `/admin/logs` jika itu merupakan endpoint existing untuk data yang sama.

Buat halaman Logs yang rapi.

Kolom minimal:

- Timestamp
- Provider
- Model
- Status
- HTTP Status
- Prompt Tokens
- Completion Tokens
- Total Tokens
- Latency
- Error Message

Jika tersedia:

- Request ID
- Client/API identifier
- API key masked

Jangan tampilkan raw API key.

==================================================
10. STATUS LOG
==================================================

Gunakan status backend yang sebenarnya.

Minimal bedakan:

SUCCESS
ERROR
BLOCKED

Contoh:

SUCCESS
HTTP 200

ERROR
HTTP 401/403/429/500/etc sesuai upstream

BLOCKED
request tidak diteruskan karena provider/model disabled/invalid sesuai behavior backend.

Jangan mengubah status hanya berdasarkan warna UI.

==================================================
11. HTTP STATUS
==================================================

Tampilkan HTTP status asli dari backend.

Contoh:

200
400
401
403
429
500

Jika `httpStatus = null`:

tampilkan `—`.

Jangan menganggap null sebagai 200.

Jangan mengarang HTTP status.

==================================================
12. LOG DETAIL
==================================================

Jika endpoint:

GET `/admin/usage/records/:index`

tersedia, gunakan endpoint tersebut untuk detail.

Buat detail view/modal/page sesuai pola UI existing.

Tampilkan:

- timestamp
- provider
- model
- status
- HTTP status
- latency
- prompt tokens
- completion tokens
- total tokens
- error message
- request ID jika tersedia
- masked client/API key jika tersedia

Credential rahasia tidak boleh ditampilkan.

==================================================
13. FILTER LOG
==================================================

Tambahkan filter menggunakan parameter yang sudah didukung backend.

Minimal:

Provider
Model
Status
Time range
Search

Jika backend mendukung:

request ID / trace ID

Gunakan query parameter backend existing.

Jangan membuat filtering palsu yang hanya bekerja di frontend jika dataset sudah dipagination backend.

==================================================
14. PAGINATION
==================================================

Gunakan:

limit
offset

sesuai endpoint existing.

Jangan mengambil seluruh Usage Logs jika jumlah record besar.

UI harus memiliki:

Previous
Next

atau pagination yang sesuai desain existing.

Tampilkan jumlah record jika backend menyediakan total.

Jika backend belum memberikan total:
- jangan membuat total palsu.

==================================================
15. SEARCH
==================================================

Jika endpoint `/admin/usage/records` mendukung search:

gunakan search backend.

Search dapat digunakan untuk:

- request ID
- trace ID
- model
- provider

sesuai parameter yang benar-benar didukung backend.

Jangan membuat query parameter baru jika backend belum mendukungnya tanpa alasan.

==================================================
16. PROVIDER FILTER
==================================================

Provider filter harus mengambil daftar provider dari provider registry/backend.

Jangan hardcode:

NVIDIA
TokenHarbor
dan provider lainnya.

Jika provider baru ditambahkan nanti, filter harus dapat mengenalinya otomatis.

==================================================
17. MODEL FILTER
==================================================

Model filter harus menggunakan model yang tersedia dari backend.

Jika memungkinkan, ketika provider dipilih:

Provider = NVIDIA

maka pilihan model hanya menampilkan model NVIDIA.

Jika Provider = TokenHarbor

maka model TokenHarbor ditampilkan.

Jangan membuat daftar model manual.

==================================================
18. DASHBOARD REFRESH
==================================================

Tambahkan refresh mechanism yang aman.

Minimal:
- tombol Refresh
- data provider refresh
- usage summary refresh
- provider breakdown refresh
- model breakdown refresh
- logs refresh

Jangan melakukan polling agresif.

Jangan membuat request berulang tanpa kontrol.

==================================================
19. LOADING STATE
==================================================

Setiap bagian dashboard harus memiliki loading state.

Contoh:

Loading providers...
Loading usage...
Loading logs...

Jangan menampilkan data kosong seolah-olah memang tidak ada data saat request masih berjalan.

==================================================
20. ERROR STATE
==================================================

Jika API gagal:

Tampilkan pesan yang jelas.

Contoh:

Failed to load providers
Failed to load usage
Failed to load logs

Jangan:
- membuat dummy data
- mengisi angka 0 palsu
- menganggap request berhasil.

Jika satu endpoint gagal:
- jangan sampai seluruh halaman crash.

==================================================
21. EMPTY STATE
==================================================

Jika benar-benar tidak ada data:

Providers:
No providers registered.

Usage:
No usage data available.

Logs:
No usage records found.

Jangan menyamakan loading dengan empty state.

==================================================
22. RESPONSIVE MOBILE UI
==================================================

Project akan digunakan dari HP.

Pastikan dashboard tetap nyaman pada layar kecil.

Untuk tabel yang lebar:
- gunakan horizontal scrolling
- atau responsive card layout
- jangan membuat teks terpotong tanpa cara melihat detail.

Provider card harus tetap mudah digunakan di HP.

Tombol Enable/Disable harus mudah ditekan.

==================================================
23. DESAIN
==================================================

Gunakan design system existing.

Jangan mengganti seluruh UI project.

Pertahankan:
- warna
- typography
- spacing
- button style
- card style
- navigation
- layout

Jika project belum memiliki komponen tertentu:
buat komponen kecil yang reusable.

Contoh:

ProviderCard
UsageSummary
UsageProviderTable
UsageModelTable
UsageLogsTable
UsageLogDetail

Hindari satu file frontend yang terlalu besar.

==================================================
24. FRONTEND API CLIENT
==================================================

Cari API client/service existing.

Gunakan client tersebut.

Jangan membuat fetch/axios client kedua jika project sudah memiliki API abstraction.

Pastikan:
- authentication admin tetap digunakan
- error handling konsisten
- base URL existing digunakan.

==================================================
25. BACKEND COMPATIBILITY
==================================================

Jangan mengubah backend hanya karena frontend membutuhkan nama field berbeda.

Sesuaikan frontend dengan response backend existing.

Jika ada mismatch field:
- audit response sebenarnya
- gunakan mapping kecil di service layer
- jangan mengubah API contract tanpa alasan.

==================================================
26. PROVIDER DISABLE + USAGE REGRESSION
==================================================

Pastikan UI tidak menghapus Usage History saat provider di-disable.

Contoh:

NVIDIA:
100 request
→ Disable NVIDIA

Usage:
tetap 100 request.

Kemudian:
Enable NVIDIA
→ request baru

Usage:
101 request.

Provider disable hanya menghentikan request baru.

==================================================
27. MODEL REGISTRY REGRESSION
==================================================

Pastikan dashboard tidak mengubah registry model.

Tetap pertahankan:

- live discovery
- provider registry
- model registry
- `/v1/models`

Provider disabled tetap dapat dikenal admin sesuai behavior existing.

Jangan menghapus model hanya karena provider disabled.

==================================================
28. BACKUP / RESTORE REGRESSION
==================================================

Jangan merusak Backup/Restore yang sudah selesai.

Pastikan dashboard tidak:
- mengubah backup schema
- menghapus usage records
- mengubah provider state secara langsung tanpa endpoint.

Provider state tetap melalui Provider Management API.

Usage tetap melalui Usage Tracking.

==================================================
29. SECURITY
==================================================

Audit UI dan API usage.

Jangan tampilkan:

- raw NVIDIA API key
- raw TokenHarbor API key
- Authorization header
- provider secret
- `.env`
- password
- private key

Jika ada:

apiKeyMasked

gunakan nilai masked tersebut.

Contoh:

`...dpP55`

atau pola masking existing.

Jangan melakukan unmask di frontend.

==================================================
30. ACCESS CONTROL
==================================================

Pastikan seluruh endpoint:

`/admin/*`

tetap membutuhkan admin authentication sesuai sistem existing.

Jangan membuat endpoint admin menjadi public.

Jangan memindahkan data usage ke endpoint `/v1/*`.

==================================================
31. PERFORMANCE
==================================================

Hindari:

- request API berulang tanpa kebutuhan
- fetch semua logs
- rendering ribuan record sekaligus
- polling agresif
- query duplikat

Gunakan pagination dan endpoint agregasi yang sudah tersedia.

==================================================
32. TEST FRONTEND
==================================================

Jika project memiliki frontend test framework, tambahkan test untuk:

1. Provider list tampil.
2. Provider status tampil.
3. Disable provider berhasil.
4. Enable provider berhasil.
5. Error disable ditampilkan.
6. Usage summary tampil.
7. Provider usage tampil.
8. Model usage tampil.
9. Logs tampil.
10. Pagination bekerja.
11. Filter provider bekerja.
12. Filter model bekerja.
13. Filter status bekerja.
14. Token tampil sesuai backend.
15. Null token tampil sebagai `—`.
16. HTTP status tampil benar.
17. Credential tidak tampil.
18. Empty state tampil.
19. Loading state tampil.
20. API error tidak membuat dashboard crash.

==================================================
33. BACKEND TEST REGRESSION
==================================================

Jalankan test internal yang relevan.

Jangan menjalankan test Gorouter.app.

Jika test suite otomatis menjalankan test Gorouter:

- skip/exclude test Gorouter
- jangan mengubah test Gorouter agar terlihat pass
- jangan menggunakan credential Gorouter
- jangan menggunakan Gorouter sebagai fallback
- laporkan test yang di-skip.

NVIDIA dan TokenHarbor.ai harus tetap didukung.

Namun Prompt 12 fokus pada Admin UI, bukan melakukan live provider testing ulang.

==================================================
34. LINT / BUILD
==================================================

Jalankan:

npm run lint
npm run build

Kemudian test internal yang relevan.

Jika `npm test` menghasilkan failure karena:

`models.test.ts`

yang membutuhkan:

`GOROUTER_API_KEY`

jangan mengubah test tersebut hanya agar pass.

Laporkan sebagai pre-existing/environment limitation jika memang bukan akibat perubahan Prompt 12.

==================================================
35. AUDIT SETELAH CODING
==================================================

Setelah coding selesai, lakukan audit:

- Provider list
- Provider enable/disable
- Provider model list
- Usage summary
- Provider usage
- Model usage
- Logs
- Filters
- Pagination
- Token display
- HTTP status
- Error handling
- Mobile UI
- Authentication
- Credential masking
- Backup compatibility
- Model registry compatibility

Pastikan tidak ada data dummy.

==================================================
36. JANGAN MELAKUKAN
==================================================

JANGAN:

- test Gorouter.app
- menggunakan Gorouter sebagai fallback
- menggunakan credential Gorouter
- membuat provider dummy
- membuat model dummy
- membuat token dummy
- membuat usage dummy
- mengestimasi token
- menghapus usage history
- menghapus provider saat disable
- mengubah model ID asli
- hardcode daftar provider
- hardcode daftar model
- membuat API client duplikat
- membuat endpoint duplikat
- mengubah backup schema tanpa kebutuhan
- menghapus test existing
- memodifikasi test Gorouter hanya agar pass
- melakukan refactor besar yang tidak diperlukan.

==================================================
37. HASIL AKHIR WAJIB DILAPORKAN
==================================================

Setelah selesai berikan laporan lengkap:

A. FILE YANG DIUBAH
- frontend files
- backend files jika ada
- test files
- documentation jika ada

B. PROVIDER MANAGEMENT
- provider list
- enable
- disable
- persistence
- model list

C. USAGE DASHBOARD
- total requests
- success
- error
- blocked
- prompt tokens
- completion tokens
- total tokens
- latency

D. PROVIDER BREAKDOWN
- provider
- requests
- success/error/blocked
- tokens
- latency

E. MODEL BREAKDOWN
- model
- provider
- requests
- tokens
- status

F. LOGS
- timestamp
- provider
- model
- status
- HTTP status
- tokens
- latency
- error
- request ID jika ada

G. FILTER
- provider
- model
- status
- search
- time range

H. PAGINATION
- limit
- offset
- previous/next

I. SECURITY
- API key masking
- Authorization header protection
- credential protection

J. RESPONSIVE
- mobile
- desktop

K. TEST
- jumlah test pass
- jumlah test fail
- test yang di-skip
- alasan failure

L. BUILD
- npm run lint
- npm run build

M. REGRESSION
Konfirmasi fitur berikut tidak rusak:

- Provider Registry
- Model Registry
- `/v1/models`
- Provider Enable/Disable
- NVIDIA
- TokenHarbor.ai
- Usage Tracking
- Usage Dashboard
- Logs
- Backup
- Restore

==================================================
HASIL YANG DIHARAPKAN
==================================================

Setelah Prompt 12 selesai, Admin Dashboard `nvidia-api` harus sudah menjadi pusat monitoring dan management:

PROVIDERS
→ lihat semua provider
→ lihat model
→ Enable/Disable

USAGE
→ total request
→ success/error/blocked
→ total token
→ latency

PROVIDERS USAGE
→ penggunaan per provider

MODELS USAGE
→ penggunaan per model

LOGS
→ detail request
→ provider
→ model
→ status
→ HTTP status
→ token
→ latency
→ error

FILTER
→ provider
→ model
→ status
→ waktu
→ search

Semua data harus berasal dari backend/API yang sebenarnya.

Tidak boleh ada data dummy atau estimasi.

Jangan mengerjakan fitur lain di luar scope ini tanpa alasan yang benar-benar diperlukan untuk integrasi.

Selesaikan implementasi, jalankan lint/build/test internal, lalu berikan laporan lengkap sesuai bagian HASIL AKHIR WAJIB DILAPORKAN.


```
# 
```
Lakukan **Final Integration Audit dan Real API Testing** pada project `nvidia-api` setelah Prompt 1, 2, dan 3 selesai.

TUJUAN:
Memastikan Provider Management, Enable/Disable Provider, Usage Tracking, Usage Dashboard, dan Logs benar-benar terintegrasi dan bekerja pada request API nyata.

1. AUDIT PROVIDER REGISTRY

* Periksa source code/provider registry yang benar-benar digunakan `nvidia-api`.
* Tampilkan/identifikasi seluruh provider yang benar-benar terdaftar.
* Jangan membuat provider atau model dummy.
* Periksa model yang benar-benar tersedia dari masing-masing provider.
* Pastikan provider disabled tidak digunakan oleh request baru.

2. AUDIT MODEL REGISTRY
   Periksa model yang tersedia dan pastikan ID model yang digunakan oleh API benar-benar sesuai dengan registry/provider.

Khusus untuk DeepSeek:

* Cari apakah `DeepSeek-V4-Flash-0371` benar-benar terdaftar.
* Jika tersedia, gunakan exact model ID yang ditemukan di source/config/provider registry.
* Jangan mengubah nama model hanya berdasarkan asumsi.
* Jika model tersebut tidak tersedia, laporkan exact model ID yang tersedia dan jangan membuat model palsu.

3. REAL API SMOKE TEST
   Lakukan testing menggunakan endpoint API yang benar-benar digunakan project.

Test minimal:

* request menggunakan provider aktif
* request menggunakan model yang valid
* request model tidak valid
* request ketika provider disabled
* enable kembali provider
* request setelah provider di-enable
* request success
* request upstream error jika dapat direproduksi dengan aman
* streaming jika endpoint mendukung streaming

4. DEEPSEEK TEST
   Jika `DeepSeek-V4-Flash-0371` benar-benar tersedia di registry dan konfigurasi:

* lakukan minimal satu smoke test request dengan model tersebut
* pastikan response berhasil
* pastikan provider dan model pada Usage Log sesuai dengan request sebenarnya
* pastikan input tokens, output tokens, dan total tokens berasal dari usage response provider jika tersedia
* pastikan latency tercatat
* pastikan status request tercatat sebagai success

Jika model tidak tersedia:

* jangan mengubah registry untuk memaksakan model tersebut
* laporkan model ID yang tersedia untuk DeepSeek V4 Flash.

5. PROVIDER DISABLE TEST
   Pastikan alur berikut bekerja:
   ACTIVE → request berhasil → DISABLE → request diblokir → ENABLE → request berhasil kembali.

Pastikan:

* blocked request tercatat sebagai `blocked`
* provider/model tetap dapat diidentifikasi jika tersedia
* tidak ada request yang lolos ke provider yang sedang disabled
* status tetap persistent setelah restart.

6. USAGE & LOG CONSISTENCY
   Bandingkan request nyata dengan log yang dihasilkan.

Pastikan setiap request memiliki data yang konsisten:

* provider
* model
* status
* HTTP status
* input tokens
* output tokens
* total tokens
* latency
* error information jika gagal
* timestamp
* API key/client identifier yang sudah dimasking

Jangan mengarang atau mengestimasi token.

7. STREAMING
   Jika API mendukung streaming:

* pastikan streaming tetap berjalan setelah Usage Tracking ditambahkan
* usage dari streaming response dicatat jika provider mengirimkannya
* logging tidak menyebabkan stream terputus
* error streaming tetap tercatat dengan benar.

8. REGRESSION AUDIT
   Pastikan fitur existing tidak rusak:

* endpoint `/v1/*`
* model discovery
* provider discovery
* request biasa
* streaming
* admin provider
* enable/disable provider
* usage
* logs

Jangan melakukan refactor besar.

9. TESTING
   Tambahkan integration/smoke test yang relevan untuk:

* provider active
* provider disabled
* provider re-enabled
* valid model
* invalid model
* usage record
* provider/model log consistency
* token usage
* streaming jika tersedia
* persistence setelah restart

Jalankan:

* npm run lint
* npm run build
* npm test

Untuk `models.test.ts`:

* jangan mengubah test hanya untuk membuatnya lulus.
* Audit penyebabnya.
* Jika memang tetap gagal karena environment/gorouter model discovery yang sudah ada sebelum Prompt 1, laporkan sebagai pre-existing failure.
* Jika audit membuktikan Prompt 1/2/3 menyebabkan failure, baru perbaiki penyebab sebenarnya.

10. HASIL AKHIR
    Berikan laporan:

* Provider yang benar-benar terdaftar
* Model yang benar-benar tersedia
* Exact model ID DeepSeek V4 Flash yang ditemukan
* Hasil test DeepSeek-V4-Flash-0371 jika tersedia
* Hasil Enable/Disable Provider
* Hasil Usage/Logs
* Hasil streaming
* File yang berubah
* Jumlah test pass/fail
* lint/build result
* status `models.test.ts`
* masalah yang masih tersisa

PENTING:
Jangan membuat data/provider/model palsu hanya agar test lulus.
Gunakan konfigurasi dan registry nyata dari project `nvidia-api`.
Jangan membocorkan API key, provider credential, Authorization header, atau secret dalam output/log.



```

# Prompt 2 — Usage Tracking.
```

Implementasikan fitur **Usage Tracking** pada project `nvidia-api`.

TUJUAN:
Mencatat dan menyediakan data penggunaan API secara akurat untuk setiap request yang diproses oleh `nvidia-api`, termasuk provider dan model yang digunakan.

FITUR:

1. Catat setiap request API yang berhasil diproses.
2. Data usage minimal mencakup:

   * timestamp
   * API key/client identifier jika sistem sudah memilikinya
   * provider
   * model
   * jumlah request
   * input/prompt tokens jika tersedia
   * output/completion tokens jika tersedia
   * total tokens jika tersedia
   * latency jika informasi tersebut sudah tersedia di sistem
   * status request (success/error)
3. Usage harus bisa dibedakan berdasarkan:

   * provider
   * model
   * API key/client
4. Jika provider di-disable, request yang ditolak tetap boleh dicatat sebagai error/blocked jika arsitektur existing mendukungnya, tetapi jangan mengubah perilaku disable provider yang sudah dibuat pada Prompt 1.
5. Jangan mengarang jumlah token. Jika provider/API tidak mengembalikan token usage, simpan sebagai null/0 sesuai pola data yang sudah digunakan project.
6. Gunakan struktur database/storage yang sudah ada. Jangan membuat database baru atau sistem storage duplikatif.
7. Pastikan pencatatan usage tidak menyebabkan request utama gagal hanya karena proses logging usage mengalami error.
8. Jangan mengubah endpoint API existing kecuali memang diperlukan untuk mengambil data usage.
9. Siapkan service/repository usage yang nantinya mudah digunakan oleh dashboard pada tahap berikutnya.
10. Tambahkan endpoint internal/admin untuk mengambil data usage jika pola API project memang menggunakan endpoint admin. Minimal sediakan data agregat:

    * total requests
    * total successful requests
    * total failed requests
    * total tokens jika tersedia
    * usage per provider
    * usage per model

TESTING:

* Tambahkan unit/integration test untuk usage tracking.
* Test request berhasil menghasilkan record usage.
* Test request gagal tidak menghasilkan data success yang salah.
* Test provider/model tercatat sesuai request sebenarnya.
* Test token usage menggunakan nilai asli dari response jika tersedia.
* Test logging usage tidak membuat request utama gagal.
* Jalankan lint/typecheck/build/test yang tersedia.

PENTING:

* Jangan menghapus atau merusak fitur existing.
* Jangan mengubah Provider Management dan Enable/Disable yang sudah dibuat pada Prompt 1 kecuali diperlukan untuk integrasi usage.
* Jangan mengerjakan UI/dashboard analytics terlebih dahulu.
* Jangan membuat mock provider.
* Jangan mengarang data usage.
* Jangan melakukan refactor besar yang tidak diperlukan.
* Pertahankan kompatibilitas dengan endpoint API yang sudah ada.

Setelah selesai, berikan ringkasan:

* File yang diubah
* Struktur data usage yang dibuat
* Endpoint/service usage yang tersedia
* Test yang ditambahkan
* Hasil build/test
* Error atau masalah yang masih ada


```
# Prompt 1 — Provider Registry + Enable/Disable
```

Audit dan implementasikan fitur **Provider Management** pada project `nvidia-api`.

TUJUAN:
Menambahkan halaman/section yang menampilkan seluruh provider yang terdaftar pada sistem NVIDIA API dan memungkinkan admin mengaktifkan atau mematikan provider.

FITUR:

1. Tampilkan daftar provider yang terdaftar.
2. Untuk setiap provider tampilkan:

   * Nama provider
   * Provider ID jika tersedia
   * Model yang tersedia
   * Status: Active / Disabled
3. Tambahkan tombol **Enable / Disable Provider**.
4. Saat provider di-disable:

   * Provider tidak boleh digunakan untuk request baru.
   * Jangan menghapus konfigurasi atau data provider.
   * Status harus tersimpan secara persistent di database/storage.
5. Saat provider di-enable kembali:

   * Provider kembali dapat digunakan tanpa konfigurasi ulang.
6. Jangan mengubah atau merusak API endpoint dan fitur yang sudah berjalan.
7. Gunakan struktur kode yang sudah ada; jangan membuat sistem provider baru yang duplikatif.
8. Pastikan state provider tetap benar setelah server restart.
9. Tambahkan validasi/error handling untuk provider yang tidak ditemukan.
10. Setelah implementasi:

* Jalankan lint/typecheck/build/test yang tersedia.
* Perbaiki error yang ditemukan.
* Audit perubahan agar tidak merusak fitur existing.

PENTING:

* Jangan menghapus fitur existing.
* Jangan melakukan refactor besar yang tidak diperlukan.
* Jangan membuat mock provider untuk menggantikan provider asli.
* Gunakan data/provider registry yang benar-benar digunakan oleh `nvidia-api`.
* Jangan lanjut mengerjakan Usage atau dashboard analytics pada tahap ini. Fokus hanya pada Provider Management dan Enable/Disable.

Setelah selesai, berikan ringkasan:

* File yang diubah
* Fitur yang berhasil dibuat
* Cara testing Enable/Disable Provider
* Hasil build/test
* Masalah yang masih ditemukan, jika ada


```
