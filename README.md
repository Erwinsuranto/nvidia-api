# nvidia-api






# 
```



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
