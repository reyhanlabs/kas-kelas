# Buku Kas Kelas (v2 — dengan Login & Tahun Ajaran)

Aplikasi pencatatan kas kelas berbasis HTML satu file (`index.html`). Sudah dilengkapi:

- **Login wajib (Firebase Authentication)** — tidak ada yang bisa membuka/mengubah data tanpa akun yang didaftarkan admin.
- **Peran (role)**: `admin` bisa menambah/mencabut akses user lain lewat menu **Kelola User**; `bendahara` bisa input & lihat data tapi tidak bisa kelola user.
- **Tahun Ajaran terpisah** — setiap tahun ajaran adalah "buku" sendiri (siswa & rekap tidak tercampur), dengan opsi salin daftar siswa saat tahun ajaran baru dimulai.
- Master Siswa, Pencatatan Pembayaran (multi-bulan sekaligus), Cetak Bukti Bayar (kwitansi A5 + stempel), Rekap Laporan (matriks 12 bulan).

## Langkah 0 — Wajib sebelum dipakai online: buat akun admin pertama

Karena Firestore Rules mewajibkan login, Anda perlu membuat **1 akun admin pertama secara manual** lewat Firebase Console (sekali saja, setelah itu admin bisa menambah user lain langsung dari dalam aplikasi):

1. Buka [Firebase Console](https://console.firebase.google.com) → project **kas-kelas-9d42b**.
2. **Build → Authentication → Get started** (jika belum aktif) → tab **Sign-in method** → aktifkan provider **Email/Password**.
3. Masih di Authentication → tab **Users** → **Add user** → isi email & password admin pertama Anda → **Add user**. Salin **User UID** yang muncul di daftar.
4. Buka **Build → Firestore Database** (buat database dulu kalau belum ada — mode production, region terdekat mis. `asia-southeast2`).
5. Buat koleksi baru bernama `users` → tambah dokumen dengan **Document ID = User UID** yang tadi disalin → isi field:
   - `email` (string) = email admin tadi
   - `role` (string) = `admin`
   - `addedAt` (number) = boleh diisi angka apa saja, mis. `0`
6. Buka **Firestore → Rules**, isi dengan aturan berikut lalu **Publish**:

   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       function isSignedIn() { return request.auth != null; }
       function myRole() {
         return exists(/databases/$(database)/documents/users/$(request.auth.uid))
           ? get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role
           : null;
       }
       function isAdmin() { return isSignedIn() && myRole() == 'admin'; }

       match /users/{uid} {
         allow read: if isSignedIn() && (isAdmin() || request.auth.uid == uid);
         allow write: if isAdmin();
       }
       match /tahunAjaran/{doc} { allow read, write: if isSignedIn(); }
       match /siswa/{doc} { allow read, write: if isSignedIn(); }
       match /pembayaran/{doc} { allow read, write: if isSignedIn(); }
       match /pengaturan/{doc} { allow read, write: if isSignedIn(); }
     }
   }
   ```

7. Buka aplikasinya → login dengan email & password admin tadi. Selesai — sekarang Anda bisa menambah user lain langsung dari menu **Kelola User** tanpa perlu masuk ke Firebase Console lagi.

> Kenapa harus manual sekali di awal? Karena membuat akun baru dari dalam aplikasi hanya boleh dilakukan oleh admin yang sudah login (supaya tidak sembarang orang bisa mendaftar sendiri). Jadi akun admin pertama perlu "ditanam" langsung lewat Console.

## Menambah user lain (setelah admin pertama ada)

Login sebagai admin → menu **Kelola User** → isi email, password sementara, pilih peran (`bendahara` atau `admin`) → **+ Tambah User**. Sampaikan email & password sementara itu ke orang yang bersangkutan secara pribadi (chat/WA, bukan lewat tempat umum). Mereka bisa mengganti password sendiri lewat "Lupa password" di layar login.

Admin juga bisa **Cabut Akses** seorang user kapan saja — akun Firebase Auth-nya tetap ada, tapi tidak bisa lagi membuka data kas kelas.

## Tahun Ajaran — cara kerjanya

- Setiap **Tahun Ajaran** (mis. "2026/2027") punya identitas sendiri: nama sekolah, nama kelas, wali kelas, bendahara, dan nominal iuran default — karena biasanya semua ini berubah setiap naik kelas.
- **Daftar siswa** dan **riwayat pembayaran** juga terpisah per tahun ajaran. Data tahun lalu tidak hilang dan tidak tercampur — tinggal pilih tahun ajarannya lewat dropdown di sidebar untuk melihat kembali.
- **Saat tahun ajaran baru dimulai**: klik **+ Tahun Ajaran Baru** di sidebar (atau di tab Pengaturan) → isi label baru (mis. "2027/2028") → jika sebagian besar siswa sama (naik kelas bersama), pilih **"Salin Daftar Siswa dari Tahun Ajaran Ini"** supaya tidak perlu input ulang satu-satu — nanti tinggal edit/hapus siswa yang pindah/lulus dan tambah siswa baru. Riwayat pembayaran **tidak** ikut disalin — tahun ajaran baru selalu mulai dari status "belum lunas" semua.
- Tab **Pengaturan → Kelola Tahun Ajaran** menampilkan semua tahun ajaran yang pernah dibuat, jumlah siswa & transaksinya, serta tombol untuk beralih melihat, mengganti label, atau menghapus (hanya bisa dihapus kalau belum ada data di dalamnya).

## Struktur data Firestore

- `users/{uid}` → `{ email, role: 'admin'|'bendahara', addedAt, addedBy }`
- `tahunAjaran/{id}` → `{ label, namaSekolah, namaKelas, waliKelas, bendahara, nominalDefault, createdAt }`
- `siswa/{id}` → `{ tahunAjaranId, noAbsen, nama, namaOrtu, noHp, nominalKhusus }`
- `pembayaran/{id}` → `{ tahunAjaranId, studentId, tahun, bulanList: [1..12], nominalPerBulan, total, tanggal, metode, keterangan, createdAt }`

## Deploy ke GitHub + Vercel

```bash
git init
git add .
git commit -m "Buku Kas Kelas v2 - login & tahun ajaran"
git branch -M main
git remote add origin https://github.com/USERNAME/kas-kelas.git
git push -u origin main
```

Di [vercel.com](https://vercel.com) → **Add New → Project** → import repo → Framework preset: **Other** (static HTML, tanpa build step) → **Deploy**.

## Cetak Bukti Bayar — Kirim WA, Export Gambar & PDF

Di tab **Cetak Bukti Bayar**, tiap transaksi punya 4 tombol aksi:

- **🖨 Cetak** — cetak langsung / simpan sebagai PDF lewat dialog print browser.
- **🖼️ Gambar** — unduh bukti sebagai file PNG.
- **📄 PDF** — unduh bukti sebagai file PDF ukuran A5.
- **💬 WA** — kirim bukti ke WhatsApp orang tua. Di HP (Android/iOS Chrome/Safari terbaru) tombol ini membuka jendela "Bagikan" bawaan sistem sehingga gambar bukti bisa langsung dipilih ke aplikasi WhatsApp beserta pesan teksnya. Jika browser tidak mendukung fitur bagikan file, gambar bukti akan otomatis terunduh dan tab WhatsApp (ke nomor **No. HP** siswa jika sudah diisi di Master Siswa) akan terbuka dengan pesan teks siap kirim — tinggal lampirkan gambar yang baru diunduh secara manual.

> Isi field **No. HP / WhatsApp Orang Tua** di Master Siswa supaya tombol WA langsung membuka chat ke nomor yang tepat.

## Mode lokal / tanpa Firebase (opsional, untuk uji coba)

Di tab Pengaturan tersedia tombol untuk sementara menonaktifkan Firebase dan memakai data lokal browser saja — berguna untuk mencoba fitur tanpa memengaruhi data online. Saat mode ini aktif, login tidak diminta.
