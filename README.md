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
