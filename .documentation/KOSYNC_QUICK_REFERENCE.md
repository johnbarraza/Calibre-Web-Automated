# Quick Reference: KOSync

## 🎯 TL;DR

**CWA has a built-in KOSync server** at:
```
http://your-cwa-instance:8083/kosync/
```

It automatically:
- ✅ Generates partial MD5 checksums for books
- ✅ Identifies books when syncing from KOReader
- ✅ Updates reading status (UNREAD → IN_PROGRESS → FINISHED)
- ✅ Syncs progress to Kobo devices
- ✅ Tracks reading history (`times_started_reading`)

**No configuration needed.** Just point KOReader to the URL with your CWA credentials.

---

## 📱 Configure in KOReader

1. Settings → Synchronization → KOSync
2. Server: `http://your-instance:8083`
3. Username: [your CWA username]
4. Password: [your CWA password]
5. Done! ✓

---

## 📡 API Endpoints

| Method | URL | Purpose |
|--------|-----|---------|
| GET | `/kosync/users/auth` | Verify credentials |
| GET | `/kosync/syncs/progress/<document>` | Get progress |
| PUT | `/kosync/syncs/progress` | Send progress |
| GET | `/kosync` | Download plugin |

**All require Basic Auth:** `Authorization: Basic base64(username:password)`

---

## 💾 Data Storage

| Database | Table | Purpose |
|----------|-------|---------|
| app.db | kosync_progress | Progress data |
| metadata.db | book_format_checksums | Book identification |
| cwa.db | ReadBook | Reading status |

---

## 🔀 About kosync-dotnet

kosync-dotnet (in `extras/` folder) is a **separate implementation**. You have three options:

### Option A: Keep Using Builtin ✅
- Current setup, no changes needed
- Fully functional

### Option B: Dual Servers ⚠️
- Run both for testing
- Configure KOReader to use either URL
- Data fragmented between servers

### Option C: Replace Builtin
- Use only kosync-dotnet
- Lose CWA integrations (book ID, ReadBook sync, Kobo sync)

**Recommendation:** Stay with builtin unless kosync-dotnet offers specific advantages.

---

## 🔍 Key Concepts

### Partial MD5
Smart checksum algorithm that:
- Reads only 1KB at 11 positions (0, 1K, 4K, 16K, 64K, 256K, 1M, 4M, 16M, 64M, 256M, 1G)
- Creates unique file identifier (~1-5ms per file)
- Doesn't modify files
- Resilient to appended data

### Automatic Book Identification
When device sends progress with partial MD5:
1. CWA searches `book_format_checksums` table
2. Finds matching book in Calibre library
3. Updates ReadBook status
4. Syncs with Kobo if applicable

If no match: Still saves progress, just without Calibre metadata.

### Reading Status Updates
```
0% → UNREAD
1-98% → IN_PROGRESS (increments times_started_reading)
99-100% → FINISHED
```

---

## 🛠️ Troubleshooting

### "Unauthorized" in KOReader
- ✓ Check username/password
- ✓ Verify user exists in CWA
- ✓ Check network connection

### Progress not syncing
- ✓ Check device has network access
- ✓ Verify server URL in KOReader
- ✓ Check CWA logs: `docker logs calibre-web-automated`

### Book not identified
- ℹ️ Normal if book wasn't imported via CWA
- ✓ Progress still saves, just without metadata
- ✓ Checksums generated on import, check metadata.db

---

## 📚 Full Documentation

See `.documentation/` folder for:
- `KOSYNC_ARCHITECTURE.md` - Complete architecture & database design
- `KOSYNC_URLS_CONFIG.md` - All endpoints & authentication details
- `KOSYNC_DOTNET_INTEGRATION.md` - Integration options with kosync-dotnet

---

## 🚀 Getting Started

1. **Verify it works:**
   ```bash
   curl -X GET \
     -H "Authorization: Basic $(echo -n 'admin:admin123' | base64)" \
     http://localhost:8083/kosync/users/auth
   ```
   Should return: `{"authorized": "OK"}`

2. **Configure KOReader:**
   - Point to: `http://your-instance:8083`
   - Use your CWA credentials

3. **Start reading:**
   - Open book in KOReader
   - Scroll through some pages
   - Sync → Should save progress

4. **Verify in CWA:**
   - Open browser → Admin → CWA Stats
   - Check `kosync_progress` count increased
   - Book should show reading status updated

---

## 📊 Performance

- **Checksum generation:** 1-5ms per file (only at import)
- **Progress sync:** <100ms per update
- **Book lookup:** <10ms (indexed query)
- **Concurrent syncs:** Supported via transaction management

---

## 🔐 Security

- **Authentication:** RFC 7617 HTTP Basic Auth
- **Transport:** Use HTTPS in production (configure reverse proxy)
- **Data:** Stored in local SQLite, never sent external
- **LDAP:** Supported with fallback to local password

---

## ⚙️ Configuration

No special config needed. Uses existing CWA settings:
- User authentication
- Database paths
- LDAP (if enabled)

Environment variables:
- `NETWORK_SHARE_MODE=true` - Disable WAL if on NFS/SMB
- `TRUSTED_PROXY_COUNT=2` - If behind multiple proxies

---

## 🎯 What Happens Behind the Scenes

```
KOReader sends:
{
  "document": "b3fb8f4f8448160365087d6ca05c7fa2",
  "percentage": 0.4567,
  "progress": "cfi(...)",
  "device": "PW5"
}
         ↓
CWA receives:
  1. Validates user credentials
  2. Saves progress to kosync_progress table
  3. Searches checksums for matching book
  4. If found:
     - Updates ReadBook.read_status
     - Updates times_started_reading counter
     - Syncs with KoboReadingState
  5. Returns response with Calibre metadata
         ↓
Device gets:
{
  "document": "...",
  "timestamp": 1699564800,
  "calibre_book_id": 42,
  "calibre_book_title": "El Quijote"
}
```

---

## 📞 Support

Check logs:
```bash
docker logs -f calibre-web-automated
```

Query databases:
```bash
# Progress data
sqlite3 /config/app.db "SELECT * FROM kosync_progress LIMIT 5;"

# Book checksums
sqlite3 /config/metadata.db "SELECT * FROM book_format_checksums LIMIT 5;"

# Reading status
sqlite3 /config/cwa.db "SELECT * FROM ReadBook WHERE user_id=1;"
```

