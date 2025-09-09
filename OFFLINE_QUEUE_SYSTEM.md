# Offline Queue System Documentation

## Overview

Sistem offline queue untuk menangani kondisi ketika koneksi internet terputus. Semua operasi sync yang gagal akan disimpan dalam antrian dan diproses kembali secara otomatis ketika koneksi tersedia.

## Cara Kerja

### Normal Operation (Internet OK)
```
User Action → API Call → Online Server → Success
```

### Internet Down (Gagal Sync)
```
User Action → API Call → Network Error → Save to Queue → Continue Local
```

### Internet Back (Recovery)
```
Background Process → Check Queue → Retry Failed Calls → Success
```

### Contoh Skenario
**Skenario 1: Internet Down Saat Order**
1. User buat order → Local save
2. Sync ke online → Network error
3. Data masuk queue → Status: "pending"
4. User bisa lanjut kerja offline
5. Internet back → Background retry → Success

**Skenario 2: Multiple Failures**
1. Order 1 → Queue (retry 1)
2. Order 2 → Queue (retry 1) 
3. Order 3 → Queue (retry 1)
4. Background process → Retry semua
5. Order 1 → Success
6. Order 2 → Still fail → Retry 2
7. Order 3 → Success

## Database Table

### pending_sync_data

Tabel untuk menyimpan operasi sync yang gagal:

```sql
- id (primary key)
- endpoint (string) - API endpoint yang dituju
- method (string) - HTTP method (GET, POST, PUT, DELETE)
- payload (json) - Data yang akan dikirim
- headers (json) - HTTP headers termasuk auth token
- error_message (text) - Pesan error saat gagal
- retry_count (int) - Jumlah percobaan ulang
- max_retries (int) - Maksimal retry (default: 3)
- status (enum) - pending, processing, completed, failed
- priority (enum) - high, medium, low
- next_retry_at (timestamp) - Waktu retry berikutnya
- created_at, updated_at (timestamp)
```

## Model

### PendingSyncData

Model dengan helper methods:

**Scopes:**
- `pending()` - Status pending
- `readyForRetry()` - Siap untuk retry
- `byPriority($priority)` - Filter by priority

**Methods:**
- `canRetry()` - Cek apakah bisa retry
- `markAsProcessing()` - Mark sebagai sedang diproses
- `markAsCompleted()` - Mark sebagai selesai
- `markAsFailed($message)` - Mark sebagai gagal
- `incrementRetry($minutes)` - Increment retry count

## Service Integration

### OnlineApiService

Service ini sudah dimodifikasi untuk auto-queue failed operations:

**Methods:**
- `makeApiCall()` - Standard call dengan queue fallback
- `makeApiCallWithoutQueue()` - Call tanpa queue (untuk read-only)
- `isOnlineConnectionAvailable()` - Check connection status

**Queue Logic:**
- Failed API calls otomatis masuk queue
- Priority ditentukan berdasarkan endpoint
- Read-only operations tidak di-queue

## Priority System

**High Priority:**
- /sync/orders
- /sync/payments
- /sync/inventory/confirm
- /sync/whatsapp/process-message

**Medium Priority:**
- /sync/inventory
- /sync/products
- /sync/customers

**Low Priority:**
- Endpoints lainnya

## Artisan Commands

### Process Pending Queue

```bash
php artisan sync:process-pending [options]

Options:
--limit=10          Maksimal record yang diproses
--priority=high     Priority level (high, medium, low, all)
--dry-run          Preview tanpa eksekusi
```

### Check Queue Status

```bash
php artisan sync:status [options]

Options:
--detailed         Breakdown detail per endpoint
--recent=24        Activity dalam N jam terakhir
```

### Cleanup Old Records

```bash
php artisan sync:cleanup [options]

Options:
--days=7           Hapus record lebih tua dari N hari
--status=completed Status yang akan dihapus (completed, failed, all)
--dry-run          Preview tanpa hapus
--force            Skip confirmation
```

## API Endpoints

### Status Check

```
GET /api/admin/biller-pos/offline-status/status
```

Response:
```json
{
  "success": true,
  "data": {
    "connection_status": {
      "is_online": false,
      "status_text": "Offline",
      "last_checked": "2025-09-09T05:31:48Z"
    },
    "sync_queue": {
      "total": 15,
      "pending": 10,
      "completed": 5,
      "failed": 0
    },
    "alerts": [
      {
        "type": "warning",
        "message": "Sistem dalam mode offline",
        "action": "Periksa koneksi internet"
      }
    ]
  }
}
```

### Queue Summary

```
GET /api/admin/biller-pos/offline-status/queue-summary
```

### Pending Items

```
GET /api/admin/biller-pos/offline-status/pending-items?limit=10
```

## Frontend Integration

### OfflineStatusIndicator Component

Component Vue di header POS yang menampilkan:
- Status online/offline
- Jumlah pending sync
- Alert notifications
- Auto refresh setiap 30 detik

Location: `resources/js/components/admin/pos/OfflineStatusIndicator.vue`

Integrated di: `resources/js/components/admin/partials/HeaderPOS.vue`

## Production Setup

### Cron Jobs

Tambahkan ke server crontab:

```bash
# Process high priority setiap 5 menit
*/5 * * * * cd /path/to/app && php artisan sync:process-pending --priority=high --limit=20

# Process semua pending setiap 15 menit  
*/15 * * * * cd /path/to/app && php artisan sync:process-pending --limit=50

# Cleanup harian jam 2 pagi
0 2 * * * cd /path/to/app && php artisan sync:cleanup --days=7 --status=completed --force
```

### Monitoring

Daily checks:
```bash
# Check status
php artisan sync:status

# Process pending jika ada
php artisan sync:process-pending --priority=high

# Lihat yang gagal
php artisan sync:status --detailed
```

### Troubleshooting

**Queue stuck:**
```bash
# Reset processing status
php artisan tinker
>>> App\Models\PendingSyncData::where('status', 'processing')->update(['status' => 'pending']);
```

**Large backlog:**
```bash
# Process bertahap
for i in {1..10}; do 
    php artisan sync:process-pending --limit=50
    sleep 10
done
```

## Retry Logic

**Exponential Backoff:**
- Retry 1: 5 menit
- Retry 2: 15 menit  
- Retry 3: 45 menit
- Retry 4: 135 menit (2+ jam)

**Max Retries:** 3-5 kali, setelah itu mark as "failed"

## Benefits

- **No Data Loss** - Semua data tersimpan
- **Seamless UX** - User tidak terganggu
- **Auto Recovery** - Retry otomatis
- **Priority System** - Data penting diprioritaskan
- **Monitoring** - Real-time status
- **Scalable** - Handle banyak data

## Configuration

Queue menggunakan database connection default, tidak perlu Redis atau external queue service.

Retry delays menggunakan exponential backoff: 1, 2, 4, 8, 16 minutes (max 60 minutes).

Timeout API calls: 10 seconds dengan 2x retry otomatis.

## Notes

System dirancang untuk reliable operation dalam kondisi koneksi internet tidak stabil. Semua critical operations akan tetap tersimpan dan diproses ulang ketika koneksi kembali normal.

Frontend indicator memberikan visual feedback kepada user tentang status sistem tanpa mengganggu workflow normal.
