# KOSync URLs and Configuration

## Server Endpoints

The CWA KOSync server runs at:

```
http://your-cwa-instance:8083/kosync/
```

---

## API Endpoints

### 1. Authentication Endpoint

**URL:** `GET /kosync/users/auth`

**Purpose:** Verify user credentials are valid

**Authentication:** HTTP Basic Auth (RFC 7617)
```
Authorization: Basic base64(username:password)
```

**Example with cURL:**
```bash
curl -X GET \
  -H "Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=" \
  http://your-instance:8083/kosync/users/auth
```

**Success Response (HTTP 200):**
```json
{
    "authorized": "OK"
}
```

**Failure Response (HTTP 401):**
```json
{
    "error": 2001,
    "message": "Unauthorized"
}
```

---

### 2. Get Progress Endpoint

**URL:** `GET /kosync/syncs/progress/<document>`

**Purpose:** Retrieve reading progress for a specific document

**Parameters:**
- `document` (URL path): Partial MD5 hash of the book file

**Authentication:** HTTP Basic Auth

**Example:**
```bash
curl -X GET \
  -H "Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=" \
  "http://your-instance:8083/kosync/syncs/progress/b3fb8f4f8448160365087d6ca05c7fa2"
```

**Response (HTTP 200):**
```json
{
    "document": "b3fb8f4f8448160365087d6ca05c7fa2",
    "progress": "cfi(/6/4[chap01]!/4/2,/1:0)",
    "percentage": 0.4567,
    "device": "PW5",
    "device_id": "kobo-pw5-abc123",
    "timestamp": 1699564800,
    "calibre_book_id": 42,
    "calibre_book_title": "El Quijote",
    "calibre_book_format": "EPUB",
    "calibre_checksum_version": "koreader"
}
```

**Notes:**
- `percentage` is returned as decimal (0.4567 = 45.67%)
- Calibre fields (`calibre_*`) only present if book was matched
- `timestamp` is Unix epoch (seconds since 1970-01-01)

---

### 3. Update Progress Endpoint

**URL:** `PUT /kosync/syncs/progress`

**Purpose:** Update reading progress from a device

**Authentication:** HTTP Basic Auth

**Request Body:**
```json
{
    "document": "b3fb8f4f8448160365087d6ca05c7fa2",
    "progress": "cfi(/6/4[chap01]!/4/2,/1:0)",
    "percentage": 0.4567,
    "device": "PW5",
    "device_id": "kobo-pw5-abc123"
}
```

**Required Fields:**
- `document`: String (1-255 chars, no colons)
- `progress`: String (1-255 chars)
- `percentage`: Number (0.0-1.0 or 0-100)
- `device`: String (1-100 chars)

**Optional Fields:**
- `device_id`: String (≤100 chars)

**Example with cURL:**
```bash
curl -X PUT \
  -H "Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=" \
  -H "Content-Type: application/json" \
  -d '{
    "document": "b3fb8f4f8448160365087d6ca05c7fa2",
    "progress": "cfi(/6/4[chap01]!/4/2,/1:0)",
    "percentage": 0.4567,
    "device": "PW5",
    "device_id": "kobo-pw5-abc123"
  }' \
  http://your-instance:8083/kosync/syncs/progress
```

**Response (HTTP 200):**
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
1. ✅ Progress saved to `kosync_progress` table
2. ✅ Book lookup attempted via partial MD5
3. ✅ `ReadBook` status updated if match found
4. ✅ `KoboReadingState` updated if applicable

---

### 4. Plugin Download Endpoint

**URL:** `GET /kosync`

**Purpose:** Provides KOReader plugin download page

**Response:** HTML page with plugin download link

**Note:** The plugin file is built during CWA Docker image creation

---

## Configuration in KOReader

### Step-by-Step Setup

1. **Open KOReader Settings**
   - Menu → Settings → Synchronization

2. **Select "KOSync"**
   - Choose: KOSync (not Kobo, not Google Drive)

3. **Configure Server URL**
   ```
   Server: http://your-instance-address:8083
   ```

4. **Enter Credentials**
   ```
   Username: [your CWA username]
   Password: [your CWA password]
   ```

5. **Save & Sync**
   - KOReader will test the connection
   - Should show "Connection successful" or similar

---

## Authentication Details

### HTTP Basic Auth (RFC 7617)

CWA uses **RFC 7617 HTTP Basic Authentication**:

```
Authorization: Basic <base64(username:password)>
```

**Encoding example:**
```bash
# Username: john
# Password: mypassword123

# Base64 encode: john:mypassword123
base64 encode "john:mypassword123"
→ am9objpteXBhc3N3b3JkMTIz

# Header:
Authorization: Basic am9objpteXBhc3N3b3JkMTIz
```

### Python Example
```python
import base64
import requests

username = "john"
password = "mypassword123"
credentials = base64.b64encode(f"{username}:{password}".encode()).decode()

headers = {
    "Authorization": f"Basic {credentials}"
}

response = requests.get(
    "http://your-instance:8083/kosync/users/auth",
    headers=headers
)
print(response.json())
```

### JavaScript Example
```javascript
const username = "john";
const password = "mypassword123";
const credentials = btoa(`${username}:${password}`);

const response = await fetch(
    "http://your-instance:8083/kosync/users/auth",
    {
        method: "GET",
        headers: {
            "Authorization": `Basic ${credentials}`
        }
    }
);

const data = await response.json();
console.log(data);
```

---

## Device Types & Formats

### Common Devices
- **Kobo Devices:** Aura, Elipsa, Sage, etc.
- **e-ink Tablets:** Remarkable, Boox, etc.
- **KOReader on Android:** KOReader app

### Progress Format Examples

**EPUB (using CFI):**
```
cfi(/6/4[chap01]!/4/2,/1:0)
```

**PDF:**
```
page=42
```

**MOBI/AZW:**
```
position=1234567
```

The exact format depends on the file type and KOReader's detection.

---

## Percentage Conversion

### Input Formats

KOReader can send percentage in two formats:

**Format 1: Decimal (0.0-1.0)**
```json
{"percentage": 0.4567}  # = 45.67%
```

**Format 2: Percentage (0-100)**
```json
{"percentage": 45.67}   # = 45.67%
```

### Automatic Detection

CWA automatically detects the format:
```python
if percentage_float <= 1.0:
    percentage_float *= 100.0  # Convert to percentage
```

So you can use either format and it will work correctly.

---

## Status Codes & Error Handling

### HTTP Status Codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 400 | Bad Request (invalid fields) |
| 401 | Unauthorized (auth failed) |
| 500 | Internal Server Error |

### Error Response Format

```json
{
    "error": <error_code>,
    "message": "<error_message>"
}
```

### Error Codes

| Code | Name | Meaning |
|------|------|---------|
| 1000 | ERROR_NO_STORAGE | Database storage error |
| 2000 | ERROR_INTERNAL | Internal server error |
| 2001 | ERROR_UNAUTHORIZED_USER | Invalid credentials |
| 2002 | ERROR_USER_EXISTS | User already exists |
| 2003 | ERROR_INVALID_FIELDS | Invalid field values |
| 2004 | ERROR_DOCUMENT_FIELD_MISSING | Document field is required |

---

## Field Validation

### Document Field
- **Type:** String
- **Required:** Yes
- **Max Length:** 255 characters
- **Constraints:** No colons (`:` reserved for internal use)
- **Example:** `b3fb8f4f8448160365087d6ca05c7fa2`

### Progress Field
- **Type:** String
- **Required:** Yes
- **Max Length:** 255 characters
- **Format:** Format-specific (CFI for EPUB, page for PDF, etc.)
- **Example:** `cfi(/6/4[chap01]!/4/2,/1:0)`

### Percentage Field
- **Type:** Number (float)
- **Required:** Yes
- **Range:** 0.0-1.0 or 0-100 (auto-detected)
- **Example:** `0.4567` or `45.67`

### Device Field
- **Type:** String
- **Required:** Yes
- **Max Length:** 100 characters
- **Example:** `PW5`, `Kobo Sage`, `Android KOReader`

### Device_id Field
- **Type:** String
- **Required:** No
- **Max Length:** 100 characters
- **Example:** `kobo-abc123`, `android-device-id`

---

## Rate Limiting

**Recommended (at reverse proxy level):**
```
/kosync/users/auth: 10 requests per minute per IP
/kosync/syncs/progress: 100 requests per minute per user
```

CWA doesn't enforce these by default, but you should configure them in your reverse proxy (nginx, Traefik, etc.)

---

## Troubleshooting

### "Unauthorized" Error
- ✅ Check username and password
- ✅ Verify user exists in CWA
- ✅ Check if LDAP is enabled (fallback auth)
- ✅ Verify Basic Auth header encoding

### "Document not found" (No book match)
- ℹ️ This is NOT an error - book identification is optional
- ℹ️ Progress is still saved, just without Calibre metadata
- ✅ Verify checksums were generated for imported books
- ✅ Check `book_format_checksums` table in metadata.db

### Connection Timeout
- ✅ Check network connectivity
- ✅ Verify CWA is running (check Docker logs)
- ✅ Check firewall rules for port 8083
- ✅ Verify reverse proxy is configured (if behind proxy)

### Database Locked
- ✅ Check for concurrent access
- ✅ Verify `NETWORK_SHARE_MODE` setting if on NFS
- ✅ Check for stuck processes locking SQLite

---

## Security Considerations

### HTTPS/TLS
- **Recommended:** Always use HTTPS in production
- **Method:** Configure reverse proxy (nginx, Traefik) with TLS
- **Why:** Basic Auth sends credentials in header

### Credentials
- **Storage:** CWA uses `check_password_hash()` for storage
- **Transmission:** Use TLS to encrypt in transit
- **LDAP:** If enabled, credentials checked against LDAP server

### Data Privacy
- **Storage Location:** `/config/app.db` (local to CWA instance)
- **Backup:** Include in regular backups
- **Access:** Only accessible via authenticated API

---

## Examples by Language

### Python
```python
import requests
import base64

url = "http://your-instance:8083/kosync/syncs/progress"
auth_str = base64.b64encode(b"username:password").decode()

headers = {
    "Authorization": f"Basic {auth_str}",
    "Content-Type": "application/json"
}

data = {
    "document": "b3fb8f4f8448160365087d6ca05c7fa2",
    "progress": "cfi(/6/4[chap01]!/4/2,/1:0)",
    "percentage": 0.4567,
    "device": "PW5",
    "device_id": "device123"
}

response = requests.put(url, json=data, headers=headers)
print(response.json())
```

### JavaScript/Fetch
```javascript
const url = "http://your-instance:8083/kosync/syncs/progress";
const auth = btoa("username:password");

const data = {
    document: "b3fb8f4f8448160365087d6ca05c7fa2",
    progress: "cfi(/6/4[chap01]!/4/2,/1:0)",
    percentage: 0.4567,
    device: "PW5",
    device_id: "device123"
};

const response = await fetch(url, {
    method: "PUT",
    headers: {
        "Authorization": `Basic ${auth}`,
        "Content-Type": "application/json"
    },
    body: JSON.stringify(data)
});

const result = await response.json();
console.log(result);
```

### cURL
```bash
curl -X PUT \
  -H "Authorization: Basic $(echo -n 'username:password' | base64)" \
  -H "Content-Type: application/json" \
  -d '{
    "document": "b3fb8f4f8448160365087d6ca05c7fa2",
    "progress": "cfi(/6/4[chap01]!/4/2,/1:0)",
    "percentage": 0.4567,
    "device": "PW5",
    "device_id": "device123"
  }' \
  http://your-instance:8083/kosync/syncs/progress
```

---

## Performance Considerations

### Checksum Calculation
- **Algorithm:** Partial MD5 (samples 1KB at 11 positions)
- **Speed:** ~1-5ms per file
- **Overhead:** Minimal (doesn't read entire file)

### Database Queries
- **Checksum Lookup:** Indexed by (checksum, version)
- **Progress Lookup:** Indexed by (user_id, document)
- **Update:** Atomic transaction with commit strategy

### Concurrent Syncs
- **Design:** Thread-safe via SQLAlchemy sessions
- **Rollback Strategy:** kosync_progress always committed first
- **Locks:** SQLite WAL mode (local disk only)

