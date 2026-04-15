# Code Review — SD-104/REV00/2025

**Project:** Cinta Zakat API (Flask / Python 3)
**Commit:** `09b2fdb` — refactor: replace positional arguments in logApiCall with a dedicated LogContext dataclass
**Branch:** DEV-4009
**Reviewer:** Claude Code (automated)
**Tanggal:** 2026-04-15
**Standard:** SD-104/REV00/2025

---

## File yang Direview

| Kategori | Jumlah File |
|---|---|
| Services | 26 file |
| Models | 13 file |
| Routes | 6 file |
| Middlewares | 2 file |
| Utils | 4 file |
| Config / Index | 2 file |
| **Total** | **53 file** |

File yang diskip: `__pycache__`, test files, `.env`, `*.json`, `*.yml`

---

## Skor & Grade

```
Skor: 75% — Grade: C
PASS: 35 | FAIL: 12 | N/A: 13
```

> Peningkatan dari review sebelumnya: **53% (Grade D) → 75% (Grade C)**
> Semua 4 Major violations dari review SD-104/REV00/2025 telah diperbaiki.

---

## Tabel Ringkasan per Kategori

| Kategori | Total | Pass | Fail | N/A | Skor |
|---|---|---|---|---|---|
| QSC1 — Ketentuan Umum | 20 | 13 | 4 | 3 | 76% |
| QSC2 — Ketentuan Class | 8 | 4 | 4 | 0 | 50% |
| QSC3 — Ketentuan Variabel | 4 | 4 | 0 | 0 | 100% |
| QSC4 — Ketentuan Fungsi | 8 | 6 | 2 | 0 | 75% |
| QSC5 — Ketentuan Komentar | 4 | 4 | 0 | 0 | 100% |
| QSC6 — Ketentuan Testing | 8 | 0 | 0 | 8 | N/A |
| QSC7 — Ketentuan Design | 5 | 3 | 1 | 1 | 75% |
| QSC8 — Ketentuan Error Handling | 3 | 1 | 1 | 1 | 50% |
| **Total** | **60** | **35** | **12** | **13** | **75%** |

| Jenis | Total |
|---|---|
| Major | 2 |
| Minor | 10 |
| **Total Pelanggaran** | **12** |

---

## Detail Pelanggaran

---

### QSC1 — Ketentuan Umum

#### ❌ [QSC1-3] Inkonsistensi camelCase — Minor

Standar proyek mewajibkan camelCase, namun sebagian file masih menggunakan `snake_case` untuk nama variabel lokal dan parameter.

📍 Lokasi: `src/models/donation_model.py`, `src/models/user_model.py`, `src/services/simba_integration.py`, `src/services/dana_simba_sync_mixin.py`

Contoh pelanggaran:
```python
# dana_simba_sync_mixin.py
user_id = ...
real_name = ...
muzaki_id = ...
campaign_id = ...
```

💡 Saran: Konsistensikan ke camelCase:
```python
userId = ...
realName = ...
muzakiId = ...
campaignId = ...
```

---

#### ❌ [QSC1-4] Nested if yang dalam — Minor

Beberapa fungsi memiliki nested if yang sangat dalam (4–6 level), melanggar prinsip avoid nested conditions.

📍 Lokasi: `src/services/dana_simba_sync_mixin.py` — fungsi `_syncToSimba()` baris ~20–480; `src/services/dana_auth_service.py` — fungsi `getOrCreateUser()`

Contoh:
```python
# dana_simba_sync_mixin.py — nested 4 level
if muzaki:
    if muzaki.get('id'):
        if not npwz:
            if user:
                # ... logika di dalam 4 level nested
```

💡 Saran: Gunakan early-return / guard clause:
```python
if not muzaki or not muzaki.get('id'):
    return _handleMuzakiNotFound(donation)
if npwz:
    return _useExistingNpwz(npwz)
```

---

#### ❌ [QSC1-6] File melebihi 500 baris — **Major**

| File | Jumlah Baris | Kelebihan |
|---|---|---|
| `src/services/miniapp_gopay_service.py` | 705 | +205 baris |
| `src/services/midtrans_service.py` | 673 | +173 baris |

📍 Lokasi: `src/services/miniapp_gopay_service.py`, `src/services/midtrans_service.py`

💡 Saran pecah file:

**miniapp_gopay_service.py:**
```
src/services/gopay_order_mixin.py     — createOrder, _prepareOrder
src/services/gopay_webhook_mixin.py   — webhook, finishPayment
src/services/gopay_query_mixin.py     — checkStatus, getHistory
```

**midtrans_service.py:**
```
src/services/midtrans_snap_mixin.py   — SNAP payment flow
src/services/midtrans_webhook_mixin.py — notification handler
src/services/midtrans_query_mixin.py  — status query
```

---

#### ❌ [QSC1-8] Baris melebihi 120 karakter — Minor

Ditemukan 20 baris yang melebihi batas 120 karakter.

📍 Lokasi terbanyak:
- `src/services/simba_integration.py` — 5 baris
- `src/services/email_service.py` — 3 baris
- `src/services/dana_webhook_mixin.py` — 3 baris (baris 44, 82, 255)
- `src/services/dana_order_mixin.py` — 2 baris (baris 201, 229)

Contoh (`dana_webhook_mixin.py:44`, 127 karakter):
```python
            if Config.DANA_ENV == 'sandbox' and (DanaWebhookMixin._uat_webhook_force_error or data.get('_uat_force_error')):
```

💡 Saran:
```python
isUatError = DanaWebhookMixin._uat_webhook_force_error or data.get('_uat_force_error')
if Config.DANA_ENV == 'sandbox' and isUatError:
```

---

### QSC2 — Ketentuan Class

#### ❌ [QSC2-2] Ukuran class terlalu besar — Minor

Beberapa class masih sangat besar meski sudah menggunakan mixin pattern.

📍 Lokasi:
- `DanaPaymentService` (via 10+ mixin) — effective size ~2.000+ baris
- `DanaAuthService` — 438 baris
- `DanaSimbaSyncMixin` — 484 baris (mendekati batas)

💡 Saran: Pecah `DanaSimbaSyncMixin` ke `SimbaMuzakiResolver` dan `SimbaTransactionSync`.

---

#### ❌ [QSC2-4] SRP — satu class satu tanggung jawab — Minor

`DanaAuthService` menangani: OAuth exchange, JWT generation, user creation, muzaki linking, DANA unbinding, dan token refresh. Ini terlalu banyak tanggung jawab.

📍 Lokasi: `src/services/dana_auth_service.py`

💡 Saran: Pisahkan ke `DanaTokenService` (OAuth/token), `DanaUserService` (user lifecycle), dan `AuthService` (JWT).

---

#### ❌ [QSC2-5] OCP — tidak ada abstraksi PaymentProvider — Minor

Menambah payment method baru (e.g. OVO, ShopeePay) mengharuskan modifikasi langsung di banyak tempat karena tidak ada interface/abstraksi `PaymentProvider`.

📍 Lokasi: `src/services/dana_payment_service.py`, `src/services/midtrans_service.py`, `src/services/miniapp_gopay_service.py`

💡 Saran:
```python
from abc import ABC, abstractmethod

class PaymentProvider(ABC):
    @abstractmethod
    def createOrder(self, data: dict) -> dict: ...
    
    @abstractmethod
    def handleWebhook(self, data: dict) -> dict: ...

class DanaPaymentService(PaymentProvider):
    def createOrder(self, data): ...
```

---

#### ❌ [QSC2-7] DIP — bergantung pada implementasi konkret — Minor

Semua service membuat instance model sendiri di `__init__`, bukan menerima via injeksi.

📍 Lokasi: Semua service file — `__init__` langsung instantiate `DonationModel()`, `UserModel()`, dll.

```python
# Pola saat ini (melanggar DIP)
class UserService:
    def __init__(self):
        self.muzakiModel = MuzakiModel()   # concrete
        self.donationModel = DonationModel()  # concrete
```

💡 Saran (dependency injection):
```python
class UserService:
    def __init__(self, muzakiModel=None, donationModel=None):
        self.muzakiModel = muzakiModel or MuzakiModel()
        self.donationModel = donationModel or DonationModel()
```

---

### QSC4 — Ketentuan Fungsi

#### ❌ [QSC4-3] Fungsi melebihi 5 statement — **Major**

Ini adalah pelanggaran yang paling pervasif. Meski `createOrder` dan `webhook` sudah dipecah, masih banyak fungsi dengan 20–100+ statements.

📍 Contoh pelanggaran terbesar:

| Fungsi | File | Est. Statements |
|---|---|---|
| `_syncToSimba()` | `dana_simba_sync_mixin.py` | ~180 |
| `seamlessLogin()` | `dana_auth_service.py` | ~45 |
| `getOrCreateUser()` | `dana_auth_service.py` | ~50 |
| `verifyOtp()` | `otp_auth_service.py` | ~35 |
| `saveTransaction()` | `midtrans_service.py` | ~30 |
| `handleNotification()` | `miniapp_gopay_service.py` | ~25 |

💡 Saran: Pecah `_syncToSimba()` menjadi:
```python
def _syncToSimba(self, donation):
    muzaki = self._resolveMuzaki(donation)
    npwz = self._resolveNpwz(muzaki, donation)
    return self._saveToSimba(donation, muzaki, npwz)
```

---

#### ❌ [QSC4-6] Satu fungsi satu tanggung jawab — Minor

Beberapa fungsi masih menggabungkan validasi, transformasi data, dan persistensi dalam satu blok.

📍 Lokasi: `src/services/otp_auth_service.py:verifyOtp()`, `src/services/dana_auth_service.py:seamlessLogin()`

💡 Saran: Pisahkan ke `_validateOtp()`, `_createSession()`, `_linkMuzaki()`.

---

### QSC7 — Ketentuan Design

#### ❌ [QSC7-5] Object membuat dependensinya sendiri — Minor

Services membuat instance model secara langsung di `__init__` bukan melalui factory atau injection container.

📍 Lokasi: Seluruh service class — `UserService`, `CampaignService`, `DanaPaymentService`, dll.

(Sama dengan QSC2-7 — duplikasi finding lintas kategori checklist.)

---

### QSC8 — Ketentuan Error Handling

#### ❌ [QSC8-1] Return code, bukan Exception — Minor

Seluruh codebase menggunakan pola `Response.error(message, code)` dan `Response.success(data)` alih-alih menggunakan Exception untuk sinyal error. Ini adalah return-code pattern.

📍 Lokasi: Semua service file — pola `return Response.error(...)`.

```python
# Pola saat ini
def getProfile(self, userId=None):
    if not muzaki:
        return Response.error("Profil tidak ditemukan", 404)  # return code
```

💡 Saran: Gunakan custom exception yang ditangkap di layer route:
```python
class ProfileNotFoundError(Exception):
    pass

def getProfile(self, userId=None):
    if not muzaki:
        raise ProfileNotFoundError("Profil tidak ditemukan")
```

---

## Observasi Tambahan (di luar checklist)

### print() masih digunakan secara masif

Ditemukan **150+ pemanggilan `print()`** tersebar di seluruh codebase. Ini bukan item checklist eksplisit namun berpengaruh signifikan pada observability di production (Cloud Run tidak menangkap `stdout` sebagai structured log).

File paling banyak:
- `dana_simba_sync_mixin.py` — ~90 `print()` calls
- `simba_integration.py` — ~50 `print()` calls
- `dana_auth_service.py` — ~12 `print()` calls
- `dana_query_detail_mixin.py` — ~15 `print()` calls

💡 Saran: Ganti semua `print()` dengan `logger.debug()` / `logger.info()` / `logger.warning()` sesuai severity.

---

## Item PASS yang Menonjol

| Item | Keterangan |
|---|---|
| QSC1-12: Tanpa duplikasi | `crypto.py` utility baru mengeliminasi duplikasi signature logic |
| QSC1-16: Gunakan constants | `MIN_DONATION`, `MAX_DONATION`, `DEFAULT_DANA_METODE_ID`, `MAX_DAYS_THRESHOLD` |
| QSC4-2: Max 3 parameter | `LogContext` dataclass menyelesaikan 7-param `logApiCall` |
| QSC3-x: Variabel jelas | Naming konsisten dan deskriptif di semua service yang baru |
| QSC5-x: Komentar | Docstring bermakna, tidak ada kode yang dikomentari |
| QSC7-1: Struktur MVC | `models/`, `services/`, `routes/`, `middlewares/` terseparasi dengan baik |
| QSC8-2: try-catch | Exception handling konsisten di seluruh codebase |

---

## Rekomendasi Akhir

**Kondisi kode saat ini sudah naik secara signifikan dari Grade D (53%) ke Grade C (75%).** Semua 4 Major violations dari review sebelumnya telah diperbaiki dengan baik — termasuk pemecahan `createOrder` dan `webhook` ke sub-fungsi, `LogContext` dataclass, shared `crypto.py`, dan constants.

Dua Major yang tersisa adalah `midtrans_service.py` dan `miniapp_gopay_service.py` yang melebihi 500 baris, serta pola fungsi panjang yang masih pervasif terutama di `dana_simba_sync_mixin.py`. Prioritas perbaikan berikutnya adalah konversi `print()` ke `logger` di `dana_simba_sync_mixin.py` dan `simba_integration.py` (150+ calls), kemudian pecah 2 file yang melebihi 500 baris.

---

*Laporan dihasilkan otomatis oleh Claude Code — 2026-04-15*
*Standard: SD-104/REV00/2025 | Commit: 09b2fdb*
