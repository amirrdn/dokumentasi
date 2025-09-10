# Cara Kerja

### Mode Normal (Internet OK)
- User melakukan transaksi → Data langsung tersimpan ke server online
- Semua operasi berjalan normal

### Mode Offline (Internet Down)
- User melakukan transaksi → Data tersimpan lokal
- Operasi sync gagal → Data masuk ke antrian (queue)
- User bisa melanjutkan kerja normal

### Mode Recovery (Internet Back)
- Sistem otomatis mendeteksi koneksi kembali
- Data di antrian diproses ulang secara otomatis
- Semua data tersinkronisasi

## Status Monitoring

### Indikator Visual
- **Hijau**: Terhubung ke server online
- **Merah**: Mode offline
- **Biru**: Sedang sinkronisasi

### Informasi yang Ditampilkan
- Status koneksi real-time
- Jumlah data pending untuk disinkronisasi
- Breakdown berdasarkan prioritas (High, Medium, Low)
- Alert sistem jika ada masalah

## Fitur Utama

### Auto Queue System
- Semua operasi gagal otomatis masuk antrian
- Prioritas berdasarkan jenis data:
  - **High**: Order, pembayaran, transaksi bank
  - **Medium**: Inventory, produk, customer
  - **Low**: Data lainnya

### Retry
- Sistem mencoba ulang dengan interval yang semakin lama
- Maksimal 3-5 percobaan sebelum ditandai gagal
- Data penting diprioritaskan

### Manual Sync
- Tombol sync manual untuk data spesifik:
  - Sync Products
  - Sync Orders  
  - Sync Customers
  - Sync Inventory
