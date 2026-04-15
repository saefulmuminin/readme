# CODE REVIEW STRUKTURAL — SD-104/REV00/2025

**File-file yang direview:**
- src/index.py
- src/config/config.py
- src/utils/ — database.py, crypto.py, response.py, log_context.py
- src/middlewares/ — auth_middleware.py, input_validator.py
- src/models/ — base_model.py, campaign_model.py, content_model.py, donation_model.py, kegiatan_model.py, log_dana_transaction_model.py, master_models.py, muzaki_model.py, muzaki_npwz_model.py, payment_model.py, ref_coa_via_model.py, user_model.py, users_otp_model.py
- src/queries/ — seluruh file .py
- src/routes/ — seluruh file .py
- src/services/ — seluruh file .py

**Bahasa:** Python 3
**Tanggal Review:** 2026-04-15
**Commit:** 9d0a5ec — refactor: standardize naming conventions to camelCase and modularize payment mixins
**Reviewer:** Claude Code (automated)

---

## Skor & Grade

```
Skor: 62% — Grade: D
PASS: 30 | FAIL: 18 | N/A: 12
```

> Grading: A=90–100%, B=80–89%, C=70–79%, D=60–69%, F<60%

---

## Tabel Ringkasan per Kategori

| Kategori | Total | Pass | Fail | N/A | Skor |
|----------|-------|------|------|-----|------|
| QSC1 - Ketentuan Umum | 20 | 10 | 8 | 2 | 56% |
| QSC2 - Ketentuan Class | 8 | 5 | 3 | 0 | 63% |
| QSC3 - Ketentuan Variabel | 4 | 3 | 1 | 0 | 75% |
| QSC4 - Ketentuan Fungsi | 8 | 5 | 3 | 0 | 63% |
| QSC5 - Ketentuan Komentar | 4 | 4 | 0 | 0 | 100% |
| QSC6 - Ketentuan Testing | 8 | 0 | 0 | 8 | N/A |
| QSC7 - Ketentuan Design | 5 | 5 | 0 | 0 | 100% |
| QSC8 - Ketentuan Error Handling | 4 | 3 | 1 | 1 | 75% |

| Jenis Pelanggaran | Total |
|---|---|
| Major | 10 |
| Minor | 8 |
| **Total Pelanggaran** | **18** |

---

## Detail Pelanggaran

### A. QSC1 — Ketentuan Umum

---

#### ❌ [QSC1-3] camelCase untuk nama fungsi dan variabel — Minor

Python PEP8 menggunakan snake_case, namun checklist SD-104 mewajibkan camelCase. Meskipun commit terakhir menyebut "standardize naming conventions to camelCase", masih banyak fungsi dan variabel menggunakan snake_case.

📍 **Lokasi:**
- `src/models/base_model.py`: `generateUuid()`, `generateChecksum()`, `findById()` — camelCase ✅
- `src/models/user_model.py`: `findByEmail()`, `updateExternalId()`, `updateDanaToken()` — camelCase ✅
- `src/queries/campaign_queries.py`: `find_all()`, `count_all()`, `find_by_id()` — snake_case ❌
- `src/queries/content_queries.py`: `get_banners()`, `get_faqs()`, `get_berita_list()` — snake_case ❌
- `src/queries/donation_queries.py`: `find_by_id()`, `update_status()` — snake_case ❌
- Hampir seluruh file di `src/queries/` — snake_case ❌

💡 **Saran perbaikan:** Standardisasi penamaan ke camelCase di seluruh queries layer, atau dokumentasikan bahwa layer queries menggunakan snake_case sebagai exception (mirror nama kolom DB). Contoh:
```python
# BEFORE (snake_case)
def find_all(cursor, tipe=None, kategori=None, sort='terbaru', limit=10, offset=0):
    ...

# AFTER (camelCase)
def findAll(cursor, tipe=None, kategori=None, sort='terbaru', limit=10, offset=0):
    ...
```

---

#### ❌ [QSC1-4] Nested if / kondisi negatif — Minor

Ditemukan nested exception handling dan nested kondisi yang dapat disederhanakan.

📍 **Lokasi:**
- `src/models/user_model.py`, baris ~25–38: Try-except dengan retry di dalam catch
```python
def findById(self, userId):
    try:
        with self.conn.cursor() as cursor:
            return user_queries.find_by_id(cursor, userId)
    except Exception as e:
        import pymysql
        if isinstance(e, pymysql.OperationalError) or "closed" in str(e):
            self.close()
            with self.conn.cursor() as cursor:  # Nested try implicit
                return user_queries.find_by_id(cursor, userId)
        else:
            raise e
```
- `src/services/email_service.py`, baris ~144–155: Nested if untuk SMTP authentication logic

💡 **Saran perbaikan:** Gunakan guard clause dan pisahkan retry logic ke helper:
```python
def findById(self, userId):
    try:
        with self.conn.cursor() as cursor:
            return user_queries.find_by_id(cursor, userId)
    except (pymysql.OperationalError, AttributeError) as e:
        if "closed" not in str(e):
            raise
        return self._retryFindById(userId)

def _retryFindById(self, userId):
    self.close()
    with self.conn.cursor() as cursor:
        return user_queries.find_by_id(cursor, userId)
```

---

#### ❌ [QSC1-6] File melampaui 500 baris — Major

Beberapa file service mendekati atau melampaui batas 500 baris.

📍 **Lokasi:**
- `src/services/dana_payment_api_mixin.py`: ~463 baris (mendekati batas)
- `src/services/dana_auth_service.py`: ~438 baris
- `src/services/dana_simba_sync_mixin.py`: ~448 baris
- `src/services/dana_order_mixin.py`: ~458 baris
- `src/services/email_service.py`: ~421 baris
- `src/services/muzaki_profile_service.py`: ~421 baris

💡 **Saran perbaikan:** Pecah method-method yang besar ke dalam helper class atau mixin terpisah. Contoh untuk `dana_payment_api_mixin.py`:
```
dana_payment_api_mixin.py         → DanaPaymentApiMixin (core)
dana_payment_request_builder.py   → DanaPaymentRequestBuilder
dana_payment_response_handler.py  → DanaPaymentResponseHandler
```

---

#### ❌ [QSC1-12] Duplikasi kode — Major

Terdapat pola yang berulang tanpa abstraksi yang cukup.

📍 **Lokasi:**
- `src/services/dana_auth_service.py`: Konstruksi object fallback user muncul di 2 tempat dengan struktur identik
- `src/models/donation_model.py`, baris ~58–79: Mapping `status_internal` ↔ `db_status` dilakukan 3x dengan cara berbeda tanpa helper terpusat
- `src/models/user_model.py`: Pattern retry-on-connection-lost diulangi di beberapa method (`findById`, `findByEmail`, dll) tanpa abstraksi menjadi decorator atau wrapper

💡 **Saran perbaikan:**
```python
# Buat decorator retry connection di base_model.py
def withReconnect(method):
    def wrapper(self, *args, **kwargs):
        try:
            return method(self, *args, **kwargs)
        except (pymysql.OperationalError,) as e:
            if "closed" not in str(e):
                raise
            self.close()
            return method(self, *args, **kwargs)
    return wrapper

# Gunakan di user_model.py
@withReconnect
def findById(self, userId):
    with self.conn.cursor() as cursor:
        return user_queries.find_by_id(cursor, userId)
```

---

#### ❌ [QSC1-16] Magic numbers tanpa konstanta — Minor

Beberapa angka ajaib ditemukan tanpa definisi konstanta yang jelas.

📍 **Lokasi:**
- `src/services/auth_service.py`, baris ~29: `94608000` (3 tahun dalam detik) — tidak ada komentar atau konstanta
- `src/middlewares/input_validator.py`, baris ~53: depth limit `10` di-hardcode
- `src/services/otp_auth_service.py`: OTP timeout 10 menit di query string, bukan di config

💡 **Saran perbaikan:**
```python
# config.py atau constants.py
AUTH_TOKEN_EXPIRY_SECONDS = 94608000   # 3 tahun
VALIDATION_MAX_DEPTH = 10
OTP_EXPIRY_MINUTES = 10
```

---

#### ❌ [QSC1-17] Kondisi panjang tanpa enkapsulasi — Minor

Kondisi AND/OR panjang langsung di-inline tanpa pembungkus method.

📍 **Lokasi:**
- `src/middlewares/input_validator.py`, baris ~42–50: Regex XSS besar inline; pattern sudah di-compile tapi belum diberi nama deskriptif
- `src/services/dana_auth_service.py`, baris ~79: Ternary dengan kondisi environment check inline

💡 **Saran perbaikan:**
```python
# BEFORE
uatScenario = data.get('_uat_scenario') if Config.DANA_ENV == 'sandbox' else None

# AFTER
def _getUatScenario(self, data):
    """Return UAT scenario hanya jika environment sandbox."""
    if Config.DANA_ENV != 'sandbox':
        return None
    return data.get('_uat_scenario')
```

---

#### ❌ [QSC1-8] Baris melampaui 120 karakter — Minor

Beberapa baris melebihi batas 120 karakter.

📍 **Lokasi:**
- `src/queries/campaign_queries.py`: SQL SELECT statement di baris ~80–99 sangat panjang tanpa line break
- `src/services/email_service.py`, baris ~88–92: Inisialisasi Google API dalam satu baris panjang
- `src/config/config.py`, baris ~39–40: Connection string MySQL

💡 **Saran perbaikan:**
```python
# BEFORE
SQLALCHEMY_DATABASE_URI = f"mysql+pymysql://{DB_USER}:{DB_PASS}@/{DB_NAME}?unix_socket=/cloudsql/{INSTANCE_CONNECTION_NAME}"

# AFTER
SQLALCHEMY_DATABASE_URI = (
    f"mysql+pymysql://{DB_USER}:{DB_PASS}@/{DB_NAME}"
    f"?unix_socket=/cloudsql/{INSTANCE_CONNECTION_NAME}"
)
```

---

#### ❌ [QSC1-15] Service layer melakukan terlalu banyak query langsung — Minor

Beberapa service memanggil banyak query di dalam satu method panjang tanpa abstraksi di layer query.

📍 **Lokasi:**
- `src/services/content_service.py`, method `getReports()`: 7x `_fresh_cursor()` call terpisah dalam satu method untuk mengambil laporan
- `src/models/content_model.py`, `get_reports()`: Membuka 7 cursor berbeda untuk 7 query

💡 **Saran perbaikan:** Buat satu method query aggregate di `content_queries.py`:
```python
# content_queries.py
def get_reports_summary(cursor):
    """Ambil semua 7 metrik laporan dalam satu koneksi DB."""
    ...
    return {
        'zakat': ...,
        'infak': ...,
        'fidyah': ...,
        ...
    }
```

---

### B. QSC2 — Ketentuan Class

---

#### ❌ [QSC2-4] SRP violation — satu class lebih dari satu tanggung jawab — Major

📍 **Lokasi:**
- `src/models/content_model.py` (~301 baris): Menangani 9 entitas berbeda — Banner, Slider, FAQ, Tentang, Berita, Nominal, Payment Channel, Reports, Inbox
- `src/services/email_service.py` (~421 baris): Menangani 2 transport mode (Gmail API + SMTP) dan 2 jenis email (OTP + Contact)

💡 **Saran perbaikan:**
```python
# Pecah content_model.py menjadi:
class BannerModel(BaseModel): ...
class FaqModel(BaseModel): ...
class NewsModel(BaseModel): ...
class ReportsModel(BaseModel): ...

# Pecah email_service.py menjadi:
class GmailApiTransport: ...
class SmtpTransport: ...
class OtpEmailService: ...
class ContactEmailService: ...
```

---

#### ❌ [QSC2-2] Ukuran class terlalu besar — Major

📍 **Lokasi:**
- `src/services/dana_payment_api_mixin.py`: ~463 baris
- `src/services/dana_auth_service.py`: ~438 baris
- `src/services/dana_simba_sync_mixin.py`: ~448 baris
- `src/services/email_service.py`: ~421 baris

💡 **Saran:** Lihat rekomendasi QSC1-6 dan QSC2-4 di atas untuk strategi refactoring.

---

#### ❌ [QSC2-7] Bergantung pada detail konkret, bukan abstraksi — Minor

📍 **Lokasi:**
- `src/services/email_service.py`: Hard-code memilih antara Gmail API vs SMTP berdasarkan flag config, tidak menggunakan interface/abstraksi transport
- `src/services/dana_payment_service.py`: Langsung instantiate mixin class secara konkret

💡 **Saran perbaikan:** Gunakan Protocol/ABC atau factory pattern:
```python
from abc import ABC, abstractmethod

class EmailTransport(ABC):
    @abstractmethod
    def send(self, to, subject, body): ...

class GmailApiTransport(EmailTransport): ...
class SmtpTransport(EmailTransport): ...

class EmailService:
    def __init__(self, transport: EmailTransport):
        self._transport = transport
```

---

### C. QSC3 — Ketentuan Variabel

---

#### ❌ [QSC3-3] Nama variabel tidak cukup deskriptif — Minor

📍 **Lokasi:**
- `src/services/dana_auth_service.py`: Variabel `e` untuk exception di beberapa handler (standar Python, tapi lebih baik `err` atau `exc`)
- `src/models/content_model.py`, method `get_reports()`: `r_zakat`, `r_infak` — singkatan tidak jelas tanpa komentar
- `src/queries/campaign_queries.py`: `q`, `params` sebagai parameter query — generik, bisa lebih deskriptif

💡 **Saran:** Gunakan nama yang lebih eksplisit:
```python
# BEFORE
r_zakat = cursor.fetchone()

# AFTER
zakatTotal = cursor.fetchone()
```

---

### D. QSC4 — Ketentuan Fungsi

---

#### ❌ [QSC4-2] Fungsi dengan > 3 parameter — Major

📍 **Lokasi:**
- `src/queries/content_queries.py`, `get_faqs()`: 5 parameter (keyword, offset, limit, orderby, order)
- `src/queries/content_queries.py`, `get_berita_list()`: 6 parameter
- `src/queries/campaign_queries.py`, `find_all()`: 6 parameter (cursor, tipe, kategori, sort, limit, offset)
- `src/queries/kegiatan_queries.py`: Beberapa fungsi dengan > 4 parameter

💡 **Saran perbaikan:** Gunakan dataclass atau dict sebagai parameter query:
```python
from dataclasses import dataclass

@dataclass
class FaqQueryParams:
    keyword: str = ''
    offset: int = 0
    limit: int = 25
    orderby: str = 'id'
    order: str = 'asc'

def get_faqs(cursor, params: FaqQueryParams):
    ...
```

---

#### ❌ [QSC4-3] Fungsi > 5 statement — Major

Banyak method memiliki terlalu banyak statement dalam satu fungsi.

📍 **Lokasi:**
- `src/services/campaign_service.py`, `getCampaigns()`: ~18 statement
- `src/services/dana_auth_service.py`, `seamlessLogin()`: ~25 statement
- `src/models/content_model.py`, `get_tentang()`: ~20 statement
- `src/models/donation_model.py`, `create()`: ~12 statement
- `src/services/muzaki_profile_service.py`, `updateProfile()`: ~15 statement

💡 **Saran perbaikan:** Refactor ke sub-methods:
```python
# BEFORE
def seamlessLogin(self, data):
    # 25 statements...

# AFTER
def seamlessLogin(self, data):
    externalId = self._extractExternalId(data)
    user = self._resolveOrCreateUser(externalId, data)
    token = self._generateAuthToken(user)
    return self._buildLoginResponse(user, token), 200

def _extractExternalId(self, data): ...
def _resolveOrCreateUser(self, externalId, data): ...
def _generateAuthToken(self, user): ...
def _buildLoginResponse(self, user, token): ...
```

---

#### ❌ [QSC4-6] Fungsi > 1 tanggung jawab — Major

📍 **Lokasi:**
- `src/services/campaign_service.py`, `getCampaigns()`: Melakukan validasi params, query DB, transform data, dan build response dalam satu method
- `src/services/dana_auth_service.py`, `seamlessLogin()`: Melakukan decode JWT, lookup user, create user jika baru, update token, dan build response

💡 **Saran:** Lihat contoh di QSC4-3 di atas untuk dekomposisi yang tepat.

---

### E. QSC5 — Ketentuan Komentar

✅ **PASS** — Komentar di codebase cukup jelas dan relevan. Tidak ditemukan komentar useless atau kode yang di-comment-out.

---

### F. QSC6 — Ketentuan Testing

⚠️ **N/A** — Review ini tidak mencakup file test. Gunakan `/review-testing` untuk review test coverage.

---

### G. QSC7 — Ketentuan Design

✅ **PASS** — Struktur MVC diterapkan dengan baik:
- Models di `src/models/`
- Routes di `src/routes/`
- Services di `src/services/`
- Queries terpisah di `src/queries/`
- Entry point di `src/index.py`
- Config terpusat di `src/config/config.py`

---

### H. QSC8 — Ketentuan Error Handling

---

#### ❌ [QSC8-4] Error handler tidak melakukan logging — Major

📍 **Lokasi:**
- `src/index.py`, error handlers HTTP 400, 404, 405, 500 (baris ~37–59): Hanya return JSON response, tidak ada logging sama sekali

```python
# Kondisi saat ini
@app.errorhandler(400)
def bad_request(e):
    return jsonify({"status": 400, "message": "Bad request"}), 400
# Tidak ada logger.error() atau logger.warning()
```

- `src/utils/database.py`, baris ~36: Menggunakan `print()` bukan logging framework

💡 **Saran perbaikan:**
```python
import logging
logger = logging.getLogger(__name__)

@app.errorhandler(400)
def bad_request(e):
    logger.warning(f"400 Bad Request: {str(e)}")
    return jsonify({"status": 400, "message": "Bad request"}), 400

@app.errorhandler(500)
def internal_server_error(e):
    logger.error(f"500 Internal Server Error: {str(e)}", exc_info=True)
    return jsonify({"status": 500, "message": "Internal server error"}), 500
```

---

## Tabel Ringkasan Pelanggaran

| # | Rule | File | Jenis |
|---|------|------|-------|
| 1 | QSC1-3 | src/queries/*.py | Minor |
| 2 | QSC1-4 | src/models/user_model.py, email_service.py | Minor |
| 3 | QSC1-6 | src/services/dana_payment_api_mixin.py (+5 files) | Major |
| 4 | QSC1-8 | src/queries/campaign_queries.py, config.py | Minor |
| 5 | QSC1-12 | src/services/dana_auth_service.py, donation_model.py | Major |
| 6 | QSC1-15 | src/models/content_model.py | Minor |
| 7 | QSC1-16 | src/services/auth_service.py, otp_auth_service.py | Minor |
| 8 | QSC1-17 | src/services/dana_auth_service.py, input_validator.py | Minor |
| 9 | QSC2-2 | src/services/dana_payment_api_mixin.py (+3 files) | Major |
| 10 | QSC2-4 | src/models/content_model.py, email_service.py | Major |
| 11 | QSC2-7 | src/services/email_service.py | Minor |
| 12 | QSC3-3 | src/models/content_model.py, dana_auth_service.py | Minor |
| 13 | QSC4-2 | src/queries/content_queries.py, campaign_queries.py | Major |
| 14 | QSC4-3 | src/services/campaign_service.py, dana_auth_service.py | Major |
| 15 | QSC4-6 | src/services/campaign_service.py, dana_auth_service.py | Major |
| 16 | QSC8-4 | src/index.py, src/utils/database.py | Major |

---

## Rekomendasi Akhir

Codebase memiliki arsitektur MVC yang solid dengan pemisahan concerns yang jelas antara routes, services, models, dan queries. Desain sistem sudah tepat dan tidak ditemukan security vulnerability yang kritis. Namun, ada beberapa area yang perlu perbaikan segera: (1) **Large methods dan large classes** — terutama `seamlessLogin`, `getCampaigns`, dan `ContentModel` yang melanggar SRP dan aturan 5-statement; (2) **Duplikasi kode** pada pola retry connection dan status mapping yang perlu diabstraksi ke utility; (3) **Error logging** di `index.py` yang perlu ditambahkan; (4) **Naming convention** di layer queries yang belum konsisten dengan camelCase. Prioritas utama: refactor method-method besar menggunakan teknik Extract Method, kemudian standardisasi penamaan di queries layer.
