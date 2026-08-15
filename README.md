# nvidia-api














# 
```



```

# 
```



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
