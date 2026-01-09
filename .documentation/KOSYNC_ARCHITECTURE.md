# KOReader Sync (KOSync) Architecture

## Overview

CWA includes a **built-in KOReader progress sync server** with automatic book identification. It allows KOReader devices to sync reading progress back to the CWA instance, which automatically identifies books and updates reading status across the Calibre-Web ecosystem.

**Official Documentation:**
```markdown
#### **KOReader Syncing (KOSync)** 📖⚡
Built-in KOReader progress sync with automatic book identification:
- **Book Identification:** Auto-generates KOReader-compatible partial MD5 checksums for all books
- **Unified Progress:** Syncs KOReader → CWA reading status → Kobo devices
- **Zero Config:** Checksums generated on startup and import, no manual setup
- **Modern Auth:** RFC 7617 HTTP Basic Auth with existing CWA accounts
- **Plugin Available:** Download from /kosync endpoint on your CWA instance
```

---

## 🔄 How It Works: The Complete Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                         KOReader Device                             │
│  (PW5, Kobo device, or any KOReader-compatible e-reader)           │
└────────────────┬────────────────────────────────────────────────────┘
                 │
                 │ 1. User reads book, makes progress
                 │ 2. Calculates partial MD5 of file (doesn't modify it)
                 │ 3. Sends: PUT /kosync/syncs/progress
                 │    {document: "b3fb8f4f...", percentage: 0.4567, ...}
                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│         CWA Instance (http://your-instance:8083/kosync/)            │
├─────────────────────────────────────────────────────────────────────┤
│ 1. Flask Blueprint: cps/progress_syncing/protocols/kosync.py        │
│    └─ authenticate_user() via RFC 7617 Basic Auth                   │
│    └─ validate request fields                                       │
│    └─ convert percentage: 0.4567 → 45.67%                           │
│                                                                     │
│ 2. Save Progress (app.db → kosync_progress table)                  │
│    ├─ user_id, document, progress, percentage                       │
│    ├─ device, device_id, timestamp                                  │
│    └─ Indexed by: (user_id, document) & document                    │
│                                                                     │
│ 3. Identify Book (metadata.db → book_format_checksums)             │
│    ├─ Query: get_book_by_checksum("b3fb8f4f...")                   │
│    ├─ Match: book_id, format, title, version                       │
│    └─ Returns calibre metadata (title, format, path)                │
│                                                                     │
│ 4. Update Reading Status (cwa.db → ReadBook)                       │
│    ├─ 0% → STATUS_UNREAD                                            │
│    ├─ 1-98% → STATUS_IN_PROGRESS                                    │
│    │          (increments times_started_reading)                    │
│    └─ 99-100% → STATUS_FINISHED                                     │
│                                                                     │
│ 5. Sync with Kobo (update KoboReadingState)                        │
│    └─ Sync reading position across Kobo ecosystem                   │
│                                                                     │
│ 6. Response to Device                                              │
│    └─ {document, timestamp, calibre_book_id, title, format}        │
└────────────────┬────────────────────────────────────────────────────┘
                 │
                 ▼
         Device updated with sync confirmation
```

---

## 🔐 Partial MD5 Checksum Algorithm

**Location:** `cps/progress_syncing/checksums/koreader.py`

### Why Partial MD5?

KOReader uses a **smart sampling algorithm** to identify books without reading the entire file:
- ✅ Fast (milliseconds, not seconds)
- ✅ Doesn't modify files
- ✅ Works with any file size
- ✅ Resilient to appended data (like PDF annotations)

### Algorithm

Samples **1024 bytes** at exponential positions:

```
Positions: 0, 1K, 4K, 16K, 64K, 256K, 1M, 4M, 16M, 64M, 256M, 1G
Formula: lshift(1024, 2*i) where i = -1 to 10
```

**Python Implementation:**
```python
def calculate_koreader_partial_md5(filepath: str) -> Optional[str]:
    """
    Returns: 32-character hexadecimal MD5 (e.g., 'b3fb8f4f8448160365087d6ca05c7fa2')
    """
    md5_hash = hashlib.md5()
    step = 1024
    sample_size = 1024

    for i in range(-1, 11):
        shift_count = 2 * i
        masked_shift = shift_count & 0x1F
        position = (step << masked_shift) & 0xFFFFFFFF
        
        f.seek(position)
        sample = f.read(sample_size)
        if sample:
            md5_hash.update(sample)
    
    return md5_hash.hexdigest()
```

**Key Feature:** Files around these sizes may see digest changes with appended data:
- 1024, 4096, 16384, 65536, 262144, 1048576, 4194304, 16777216, 67108864, 268435456, 1073741824

---

## 📊 Database Architecture

Three **separate SQLite databases**, never consolidate:

### 1. `app.db` → `kosync_progress` Table
**Stores:** Reading progress from KOReader devices

```sql
CREATE TABLE kosync_progress (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,           -- FK to user(id)
    document TEXT NOT NULL,             -- Partial MD5 hash
    progress TEXT NOT NULL,             -- Position (CFI for EPUB, etc.)
    percentage REAL NOT NULL,           -- 0-100
    device TEXT NOT NULL,               -- "KOReader", "PW5", etc.
    device_id TEXT,                     -- Device identifier
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES user(id)
);

-- Indexes for fast lookup
CREATE INDEX idx_kosync_user_document ON kosync_progress(user_id, document);
CREATE INDEX idx_kosync_document ON kosync_progress(document);
```

### 2. `metadata.db` → `book_format_checksums` Table
**Stores:** Checksums for all imported books (history tracking)

```sql
CREATE TABLE book_format_checksums (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    book INTEGER NOT NULL,              -- FK to books(id)
    format TEXT NOT NULL COLLATE NOCASE,-- "EPUB", "AZW3", etc.
    checksum TEXT NOT NULL,             -- Partial MD5
    version TEXT NOT NULL DEFAULT 'koreader', -- Algorithm version
    created TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (book) REFERENCES books(id)
);

-- Optimized indexes
CREATE INDEX idx_checksum ON book_format_checksums(checksum);
CREATE INDEX idx_checksum_version ON book_format_checksums(checksum, version);
CREATE INDEX idx_book_format ON book_format_checksums(book, format);
CREATE INDEX idx_created ON book_format_checksums(created);
```

**Why History Tracking?**
- Allows sync with ANY version of a book file that exists
- Latest checksum determined by `created` timestamp
- No distinction between 'library' and 'OPDS' checksums

### 3. `cwa.db` → `ReadBook` + `KoboReadingState`
**Stores:** Reading status, times started reading, Kobo sync state

```python
class ReadBook:
    user_id
    book_id
    read_status  # UNREAD, IN_PROGRESS, FINISHED
    times_started_reading
    last_time_started_reading
    kobo_reading_state  # FK to KoboReadingState
        └── current_bookmark
            └── progress_percent
```

---

## 🌐 API Endpoints

### Endpoints

All endpoints use **HTTP Basic Authentication (RFC 7617)**

```python
Authorization: Basic base64(username:password)
```

#### 1. `GET /kosync/users/auth`
**Purpose:** Verify authentication

**Request:**
```http
GET /kosync/users/auth HTTP/1.1
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

**Response (Success):**
```json
{
    "authorized": "OK"
}
```

**Response (Failure):**
```json
{
    "error": 2001,
    "message": "Unauthorized"
}
```

---

#### 2. `GET /kosync/syncs/progress/<document>`
**Purpose:** Retrieve reading progress for a document

**Request:**
```http
GET /kosync/syncs/progress/b3fb8f4f8448160365087d6ca05c7fa2 HTTP/1.1
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

**Response:**
```json
{
    "document": "b3fb8f4f8448160365087d6ca05c7fa2",
    "progress": "cfi(/6/4[chap01]!/4/2,/1:0)",
    "percentage": 0.4567,
    "device": "PW5",
    "device_id": "123456",
    "timestamp": 1699564800,
    "calibre_book_id": 42,
    "calibre_book_title": "El Quijote",
    "calibre_book_format": "EPUB",
    "calibre_checksum_version": "koreader"
}
```

**Notes:**
- Percentage returned as **decimal fraction** (0.4567 = 45.67%)
- Calibre fields only present if book was matched

---

#### 3. `PUT /kosync/syncs/progress`
**Purpose:** Update reading progress from device

**Request:**
```json
{
    "document": "b3fb8f4f8448160365087d6ca05c7fa2",
    "progress": "cfi(/6/4[chap01]!/4/2,/1:0)",
    "percentage": 0.4567,
    "device": "PW5",
    "device_id": "123456"
}
```

**Conversion Logic:**
- If `percentage ≤ 1.0`: Treated as decimal, multiplied by 100
- If `percentage > 1.0`: Treated as percentage, used as-is
- Stored in database as **0-100 range**

**Response:**
```json
{
    "document": "b3fb8f4f8448160365087d6ca05c7fa2",
    "timestamp": 1699564800,
    "calibre_book_id": 42,
    "calibre_book_title": "El Quijote",
    "calibre_book_format": "EPUB",
    "calibre_checksum_version": "koreader"
}
```

**Side Effects:**
1. ✅ Saves progress in `kosync_progress`
2. ✅ Searches for matching book by checksum
3. ✅ Updates `ReadBook.read_status` if matched:
   - 0% → UNREAD
   - 1-98% → IN_PROGRESS (increments `times_started_reading`)
   - 99-100% → FINISHED
4. ✅ Updates `KoboReadingState.progress_percent`

---

#### 4. `GET /kosync`
**Purpose:** Plugin download page

**Returns:** HTML page with KOReader plugin download link

---

## 🔧 Authentication Methods

### HTTP Basic Auth (RFC 7617)
```
Authorization: Basic base64(username:password)
```

### LDAP Support
If `config.config_login_type == LOGIN_LDAP`:
1. Try LDAP authentication first
2. Fall back to local password if LDAP fails

### Password Verification
- Uses `check_password_hash()` for constant-time comparison
- Case-insensitive username lookup
- Validates credential format before DB lookup

---

## 🎯 Automatic Book Identification

### Process

1. **Device sends:** `document: "b3fb8f4f8448160365087d6ca05c7fa2"`

2. **Query book_format_checksums:**
   ```python
   query = BookFormatChecksum.query\
       .join(Books)\
       .filter(BookFormatChecksum.checksum == document_checksum)\
       .order_by(BookFormatChecksum.created.desc())\
       .first()
   ```

3. **Returns:** `(book_id, format, title, path, version)`

4. **If matched:** Update ReadBook and KoboReadingState

5. **If not matched:** Still save progress, just without book metadata

### Checksum Generation Timing

Checksums are generated **automatically**:
- ✅ At CWA startup (for existing library)
- ✅ On book import
- ✅ After format conversion
- ✅ After EPUB fixing
- ✅ After metadata enforcement

**Zero manual configuration required.**

---

## 🔄 Reading Status Updates

### Status Transitions

```
UNREAD (0%)
   ↓
IN_PROGRESS (1-98%) ← times_started_reading increments here
   ↓
FINISHED (99-100%)
```

### Database Changes on Sync

```python
# If new ReadBook record doesn't exist:
book_read = ReadBook(
    user_id=user.id,
    book_id=book_id,
    read_status=new_status
)
if new_status == STATUS_IN_PROGRESS:
    book_read.times_started_reading = 1
    book_read.last_time_started_reading = datetime.now()

# If existing record:
if new_status != old_status and new_status == STATUS_IN_PROGRESS:
    book_read.times_started_reading += 1
    book_read.last_time_started_reading = datetime.now()

book_read.read_status = new_status
book_read.last_modified = datetime.now()

# Update Kobo bookmarks
if book_read.kobo_reading_state:
    book_read.kobo_reading_state.current_bookmark.progress_percent = percentage
```

---

## 🛡️ Data Integrity & Error Handling

### Critical Commit Strategy

```python
# Step 1: ALWAYS save kosync_progress first (NEVER rollback)
ub.session.commit()  # ✓ Sync data is safe

# Step 2: Try to update ReadBook (can rollback if fails)
try:
    update_book_read_status(user.id, book_id, percentage)
    ub.session.commit()
except SQLAlchemyError:
    ub.session.rollback()  # Only affects ReadBook, not kosync_progress
```

**Why?** Sync location data must always be persisted, even if reading status updates fail.

### Validation

- **Document:** Non-empty, no colons, ≤255 characters
- **Progress:** Non-empty string, ≤255 characters
- **Percentage:** 0.0-100.0 or 0.0-1.0 (auto-detected)
- **Device:** Non-empty, ≤100 characters
- **Device ID:** ≤100 characters (optional)

### Error Codes

| Code | Meaning |
|------|---------|
| 1000 | ERROR_NO_STORAGE |
| 2000 | ERROR_INTERNAL |
| 2001 | ERROR_UNAUTHORIZED_USER |
| 2002 | ERROR_USER_EXISTS |
| 2003 | ERROR_INVALID_FIELDS |
| 2004 | ERROR_DOCUMENT_FIELD_MISSING |

---

## 📁 File Locations

```
/config/
├── app.db              ← kosync_progress table
├── metadata.db         ← book_format_checksums table (Calibre)
├── cwa.db              ← ReadBook & KoboReadingState
└── ...
```

---

## 🚀 Configuration

### Environment Variables

None specific to KOSync. Uses existing CWA authentication.

### Settings

KOSync behavior controlled by:
- `config.config_login_type` - LDAP vs local auth
- User permissions (reading/writing)
- Database configuration

---

## 🔌 Integration Points

### With Kobo Sync
- `ReadBook` records synchronized between KOReader and Kobo devices
- `KoboReadingState.current_bookmark.progress_percent` updated on each sync

### With OPDS
- Books from OPDS feeds can be synced if checksums are stored

### With OAuth2/OIDC
- KOSync uses same user authentication as CWA
- Works with LDAP fallback

---

## ⚠️ Known Limitations

1. **Percentage Detection:** Files around 1K, 4K, 16K, etc. may see digest changes with appended data
2. **Network Shares:** WAL mode disabled when `NETWORK_SHARE_MODE=true`
3. **Device Identification:** Based on `device_id` sent by client, not guaranteed unique
4. **Offline Sync:** KOReader must have network access to CWA instance

---

## 📚 Related Files

- **Main Implementation:** `cps/progress_syncing/protocols/kosync.py` (750 lines)
- **Checksum Manager:** `cps/progress_syncing/checksums/manager.py`
- **Partial MD5:** `cps/progress_syncing/checksums/koreader.py`
- **Models:** `cps/progress_syncing/models.py`
- **Registration:** `cps/main.py` (line 31: `app.register_blueprint(kosync)`)
- **Database Init:** `cps/ub.py` (line 717)

