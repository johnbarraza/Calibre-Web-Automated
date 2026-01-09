# KOSync Development & Implementation Details

## Code Organization

```
cps/
├── progress_syncing/
│   ├── __init__.py                    # Exports & initialization
│   ├── models.py                      # Database models & table setup
│   │
│   ├── checksums/
│   │   ├── __init__.py
│   │   ├── koreader.py               # Partial MD5 algorithm (90 lines)
│   │   └── manager.py                # Checksum storage & retrieval
│   │
│   └── protocols/
│       └── kosync.py                 # KOSync server (750 lines)
│
├── main.py                            # Blueprint registration (line 31)
├── ub.py                              # Table initialization (line 717)
└── db.py                              # Database models
```

---

## File-by-File Breakdown

### 1. `cps/progress_syncing/checksums/koreader.py`

**Purpose:** Implements KOReader's partial MD5 algorithm

**Key Function:**
```python
def calculate_koreader_partial_md5(filepath: str) -> Optional[str]:
    """
    Calculate partial MD5 hash using KOReader's sampling algorithm.
    
    Returns: 32-char hex string (e.g., 'b3fb8f4f8448160365087d6ca05c7fa2')
    """
```

**Algorithm Details:**
```python
# Sample 1024 bytes at these positions:
positions = [0, 1K, 4K, 16K, 64K, 256K, 1M, 4M, 16M, 64M, 256M, 1G]

# Calculation (matches KOReader's LuaJIT implementation):
for i in range(-1, 11):
    shift_count = 2 * i
    masked_shift = shift_count & 0x1F    # LuaJIT: 5-bit mask
    position = (step << masked_shift) & 0xFFFFFFFF  # 32-bit unsigned
    
    f.seek(position)
    md5_hash.update(f.read(1024))

return md5_hash.hexdigest()
```

**Error Handling:**
```python
# Returns None if:
# - File doesn't exist
# - File can't be read (permission error)
# - Any unexpected error occurs
```

---

### 2. `cps/progress_syncing/checksums/manager.py`

**Purpose:** Manage checksum storage and retrieval

**Key Functions:**

```python
def store_checksum(
    book_id: int,
    book_format: str,
    checksum: str,
    version: str = CHECKSUM_VERSION,
    db_connection=None
) -> bool:
    """Store a checksum without deduplication."""
    # Checks for exact duplicate, but keeps all versions
    # All checksums kept indefinitely for history
```

```python
def calculate_and_store_checksum(
    book_id: int,
    book_format: str,
    file_path: str,
    db_connection=None
) -> Optional[str]:
    """Calculate checksum from file and store it."""
    # Returns the checksum string
```

```python
def get_latest_checksum(
    book_id: int,
    book_format: str
) -> Optional[str]:
    """Get most recent checksum (by created timestamp)."""
```

```python
def get_checksum_history(
    book_id: int,
    book_format: str
) -> List[Tuple[str, datetime, str]]:
    """Get complete checksum history for a book."""
    # Returns: [(checksum, created, version), ...]
```

---

### 3. `cps/progress_syncing/models.py`

**Purpose:** Database models and table initialization

**Models:**

```python
class BookFormatChecksum(CalibreBase):
    """Stored in metadata.db"""
    __tablename__ = 'book_format_checksums'
    
    id = Column(Integer, primary_key=True)
    book = Column(Integer, ForeignKey('books.id'))
    format = Column(String, nullable=False)
    checksum = Column(String, nullable=False)
    version = Column(String, default='koreader')
    created = Column(TIMESTAMP, default=current_timestamp)


class KOSyncProgress(AppBase):
    """Stored in app.db"""
    __tablename__ = 'kosync_progress'
    
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey('user.id'))
    document = Column(String, nullable=False)
    progress = Column(String, nullable=False)
    percentage = Column(Float, nullable=False)
    device = Column(String, nullable=False)
    device_id = Column(String)
    timestamp = Column(TIMESTAMP, default=current_timestamp)
```

**Table Initialization:**

```python
def ensure_calibre_db_tables(conn):
    """Create book_format_checksums table if needed"""
    ensure_checksum_table(conn)

def ensure_app_db_tables(conn):
    """Create kosync_progress table if needed"""
    ensure_kosync_progress_table(conn)
```

---

### 4. `cps/progress_syncing/protocols/kosync.py`

**Purpose:** Flask Blueprint implementing KOSync protocol (750 lines)

**Architecture:**
```python
kosync = Blueprint('kosync', __name__)
# Routes registered without url_prefix
# Results in: /kosync/users/auth, /kosync/syncs/progress, etc.
```

**Main Functions:**

#### Authentication
```python
def authenticate_user() -> Optional[ub.User]:
    """
    Authenticate via HTTP Basic Auth (RFC 7617).
    
    Returns User object or None
    """
    # 1. Extract Authorization header
    # 2. Decode base64 credentials
    # 3. Query user (case-insensitive)
    # 4. Try LDAP if enabled
    # 5. Fall back to local password
    # 6. Return user or None
```

#### Book Identification
```python
def get_book_by_checksum(
    document_checksum: str,
    version: str = None
) -> Tuple[int, str, str, str, str]:
    """
    Lookup book by checksum.
    
    Query:
        SELECT book, format, version, title, path
        FROM book_format_checksums
        JOIN books ON ...
        WHERE checksum = ?
        ORDER BY created DESC
    
    Returns: (book_id, format, title, path, version) or (None, None, ...)
    """
```

#### Status Updates
```python
def update_book_read_status(
    user_id: int,
    book_id: int,
    percentage: float
) -> None:
    """
    Update ReadBook status based on percentage.
    
    Logic:
        if percentage >= 99.0:
            status = STATUS_FINISHED
        elif percentage > 0:
            status = STATUS_IN_PROGRESS
            if old_status != IN_PROGRESS:
                times_started_reading += 1
        else:
            status = STATUS_UNREAD
    
    Also updates KoboReadingState if exists
    """
```

#### Response Handling
```python
def enrich_response_with_book_info(
    response_data: Dict,
    document_checksum: str
) -> Tuple[Dict, int, str, str, str]:
    """
    Add Calibre metadata to response if book found.
    
    Adds to response:
        - calibre_book_id
        - calibre_book_title
        - calibre_book_format
        - calibre_checksum_version
    """
```

#### Error Handling
```python
def handle_sync_error(error: KOSyncError) -> tuple:
    """Convert KOSyncError to JSON response"""

@kosync.errorhandler(400)
def handle_bad_request(error):
    """HTTP 400 handler"""

@kosync.errorhandler(401)
def handle_unauthorized(error):
    """HTTP 401 handler"""

@kosync.errorhandler(500)
def handle_internal_error(error):
    """HTTP 500 handler"""
```

**Routes:**

```python
@kosync.route("/kosync", methods=["GET"])
def plugin_download():
    """Serve plugin download page"""

@kosync.route("/kosync/users/auth", methods=["GET"])
@csrf.exempt
def authenticate():
    """Verify credentials"""

@kosync.route("/kosync/syncs/progress/<document>", methods=["GET"])
@csrf.exempt
def get_progress(document: str):
    """Get reading progress"""

@kosync.route("/kosync/syncs/progress", methods=["PUT"])
@csrf.exempt
def update_progress():
    """Update reading progress (main endpoint)"""
```

**Validation:**
```python
def is_valid_field(field: Any) -> bool:
    """Non-empty string"""

def is_valid_key_field(
    field: Any,
    max_length: int = 255
) -> bool:
    """Non-empty string, no colons, within length"""
```

**Constants:**
```python
MAX_DOCUMENT_LENGTH = 255
MAX_PROGRESS_LENGTH = 255
MAX_DEVICE_LENGTH = 100
MAX_DEVICE_ID_LENGTH = 100

ERROR_NO_STORAGE = 1000
ERROR_INTERNAL = 2000
ERROR_UNAUTHORIZED_USER = 2001
ERROR_USER_EXISTS = 2002
ERROR_INVALID_FIELDS = 2003
ERROR_DOCUMENT_FIELD_MISSING = 2004
```

---

## Integration Points

### 1. Blueprint Registration (`cps/main.py`)

```python
from .progress_syncing.protocols.kosync import kosync

# Later in main():
app.register_blueprint(kosync)  # Register without url_prefix
```

### 2. Database Initialization (`cps/ub.py`)

```python
from cps.progress_syncing.models import KOSyncProgress

# In init function:
if not engine.dialect.has_table(engine.connect(), "kosync_progress"):
    KOSyncProgress.__table__.create(bind=engine)
```

### 3. User Authentication

Uses existing CWA user system:
```python
from cps import ub

user = ub.session.query(ub.User).filter(
    func.lower(ub.User.name) == username.lower()
).first()

if user and check_password_hash(user.password, password):
    # Authentication successful
```

### 4. Reading Status Updates

Uses existing CWA models:
```python
from cps import ub

book_read = ub.session.query(ub.ReadBook).filter(
    ub.ReadBook.user_id == user_id,
    ub.ReadBook.book_id == book_id
).first()

if book_read:
    book_read.read_status = new_status
    book_read.times_started_reading += 1
    book_read.last_modified = datetime.now()
```

---

## Data Flow Diagrams

### Checksum Generation (at import)

```
Book imported
    ↓
ingest_processor.py detects import
    ↓
calculate_and_store_checksum(book_id, format, filepath)
    ↓
calculate_koreader_partial_md5(filepath)
    ↓
store_checksum(book_id, format, checksum, 'koreader')
    ↓
Stored in metadata.db.book_format_checksums
```

### Progress Update (from device)

```
Device sends PUT /kosync/syncs/progress
    ↓
update_progress()
    ├─ authenticate_user() → verify credentials
    ├─ Validate request fields
    ├─ Save to kosync_progress (COMMIT HERE)
    ├─ get_book_by_checksum() → lookup in metadata.db
    │  └─ If found:
    │     ├─ update_book_read_status()
    │     │  ├─ Update ReadBook
    │     │  └─ Update KoboReadingState
    │     └─ (COMMIT)
    │  └─ If not found:
    │     └─ (still success, just no metadata)
    └─ Return enriched response
```

---

## Performance Considerations

### Database Indexes

**kosync_progress table:**
```sql
CREATE INDEX idx_kosync_user_document ON kosync_progress(user_id, document);
CREATE INDEX idx_kosync_document ON kosync_progress(document);
```

**book_format_checksums table:**
```sql
CREATE INDEX idx_checksum ON book_format_checksums(checksum);
CREATE INDEX idx_checksum_version ON book_format_checksums(checksum, version);
CREATE INDEX idx_book_format ON book_format_checksums(book, format);
CREATE INDEX idx_created ON book_format_checksums(created);
```

### Query Performance

- Checksum lookup: O(1) via index on `checksum`
- Progress lookup: O(1) via index on `(user_id, document)`
- Update: Atomic transaction via SQLAlchemy

### File Processing

- Partial MD5: ~1-5ms (1KB × 11 samples)
- Async: Can be queued as background task
- No bottleneck: Many imports can be concurrent

---

## Error Handling Strategy

### Validation Errors
```python
if not is_valid_field(progress):
    raise KOSyncError(ERROR_INVALID_FIELDS, "Invalid progress")
```

### Authentication Errors
```python
user = authenticate_user()
if not user:
    raise KOSyncError(ERROR_UNAUTHORIZED_USER, "Unauthorized")
```

### Database Errors
```python
try:
    ub.session.commit()
except SQLAlchemyError as e:
    log.error(f"Database error: {e}")
    ub.session.rollback()
    raise KOSyncError(ERROR_INTERNAL, "Database error")
```

### Graceful Degradation
```python
# Always save progress
ub.session.commit()  # ✓ Sync data safe

# Try to update ReadBook (can fail)
try:
    update_book_read_status(...)
    ub.session.commit()
except:
    ub.session.rollback()  # Only affects ReadBook
    # Progress is still saved
```

---

## Testing Approach

### Manual Testing with cURL

```bash
# 1. Auth test
curl -X GET \
  -H "Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=" \
  http://localhost:8083/kosync/users/auth

# 2. Get progress (new book, should be empty)
curl -X GET \
  -H "Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=" \
  http://localhost:8083/kosync/syncs/progress/test123

# 3. Update progress
curl -X PUT \
  -H "Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=" \
  -H "Content-Type: application/json" \
  -d '{"document":"test123","progress":"pos1","percentage":0.25,"device":"test"}' \
  http://localhost:8083/kosync/syncs/progress

# 4. Verify it was saved
curl -X GET \
  -H "Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=" \
  http://localhost:8083/kosync/syncs/progress/test123
```

### Database Inspection

```bash
# Check if tables exist
sqlite3 /config/app.db ".tables"

# View kosync_progress table
sqlite3 /config/app.db "SELECT * FROM kosync_progress;"

# View book checksums
sqlite3 /config/metadata.db "SELECT id, book, format, checksum, created FROM book_format_checksums LIMIT 10;"

# Check ReadBook updates
sqlite3 /config/cwa.db "SELECT user_id, book_id, read_status, times_started_reading FROM ReadBook WHERE user_id=1;"
```

### Log Inspection

```bash
docker logs calibre-web-automated | grep kosync
docker logs calibre-web-automated | grep "Book authenticated successfully"
```

---

## Future Improvements

1. **Checksum Versioning:** Support multiple algorithms
2. **Sync Conflict Resolution:** Last-write-wins vs. merge strategies
3. **Bandwidth Optimization:** Only sync deltas
4. **Offline Support:** Local caching for devices
5. **Multi-Device:** Better device identification/preferences
6. **History Tracking:** Full audit log of sync events
7. **Performance:** Batch updates, connection pooling

