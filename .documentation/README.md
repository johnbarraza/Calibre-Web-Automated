# KOSync Documentation Index

Welcome to the KOReader Sync (KOSync) documentation. This folder contains comprehensive guides for understanding and working with CWA's KOSync implementation.

## 📚 Documentation Files

### 1. **KOSYNC_QUICK_REFERENCE.md** ⭐ START HERE
**For:** Everyone
- Quick overview of what KOSync is
- Basic setup in 30 seconds
- Common troubleshooting
- Key concepts explained simply

👉 **Read this first if you're new to KOSync**

---

### 2. **KOSYNC_ARCHITECTURE.md**
**For:** Developers, Advanced Users
- Complete system architecture
- Database design (3 databases explained)
- How partial MD5 checksums work
- Automatic book identification process
- Reading status update logic
- Data integrity & error handling

👉 **Read this to understand how everything works**

---

### 3. **KOSYNC_URLS_CONFIG.md**
**For:** Developers, Integration Specialists
- All API endpoints documented
- Request/response examples
- Authentication details (RFC 7617 Basic Auth)
- Field validation rules
- Code examples (Python, JavaScript, cURL)
- Troubleshooting guide

👉 **Read this to integrate or debug**

---

### 4. **KOSYNC_DOTNET_INTEGRATION.md**
**For:** Developers Wanting to Integrate kosync-dotnet

**Important:** kosync-dotnet is an **external .NET implementation** of a KOSync server. It is **NOT currently integrated into CWA**. This document explains:
- What kosync-dotnet is (separate project)
- Three integration options (A, B, C)
- What adaptations are needed
- Decision tree for choosing approach
- Migration considerations

👉 **Read this if you want to integrate kosync-dotnet with CWA**

---

### 5. **KOSYNC_DEVELOPMENT.md**
**For:** Core Developers
- Code organization & file breakdown
- Function-by-function implementation details
- Integration points with CWA
- Data flow diagrams
- Performance considerations
- Testing approaches
- Future improvement ideas

👉 **Read this if you're modifying the code**

---

## 🎯 Quick Navigation

### By Use Case

**"I just want to use KOSync"**
1. Read: KOSYNC_QUICK_REFERENCE.md
2. Configure KOReader with your CWA URL
3. Done!

**"I want to understand how it works"**
1. KOSYNC_QUICK_REFERENCE.md (overview)
2. KOSYNC_ARCHITECTURE.md (deep dive)
3. KOSYNC_URLS_CONFIG.md (API details)

**"I want to integrate with kosync-dotnet"**
1. KOSYNC_QUICK_REFERENCE.md (context)
2. KOSYNC_DOTNET_INTEGRATION.md (integration options)
3. KOSYNC_ARCHITECTURE.md (understand CWA's current implementation)
4. KOSYNC_DEVELOPMENT.md (code details for adaptation)

**"I want to modify the code"**
1. KOSYNC_ARCHITECTURE.md (understand system)
2. KOSYNC_DEVELOPMENT.md (code details)
3. KOSYNC_URLS_CONFIG.md (API contract)
4. Source code: `cps/progress_syncing/`

**"Something is broken"**
1. KOSYNC_URLS_CONFIG.md → Troubleshooting section
2. KOSYNC_DEVELOPMENT.md → Testing approaches
3. Check logs: `docker logs calibre-web-automated | grep kosync`
4. Query databases: See testing section

---

## 📊 Key Information Summary

### What is KOSync?
A reading progress sync system for KOReader devices that:
- Stores reading position (page/location in book)
- Identifies books automatically
- Tracks reading status (reading/finished)
- Syncs across Kobo devices
- Integrates with CWA's library

### Where Does It Run?
```
http://your-cwa-instance:8083/kosync/
```

### What Databases Does It Use?
1. `app.db` → `kosync_progress` (progress data)
2. `metadata.db` → `book_format_checksums` (book identification)
3. `cwa.db` → `ReadBook` (reading status)

### How Does It Identify Books?
Using **partial MD5 checksums**:
- Samples 1KB at strategic file positions
- Fast (~1-5ms per file)
- Doesn't modify files
- Works with any file size

### How Do I Configure It?
Nothing! It works automatically:
- ✅ Just point KOReader to `http://your-instance:8083`
- ✅ Use your CWA credentials
- ✅ Checksums generated on import

### Can I Use kosync-dotnet Instead?
**kosync-dotnet is an external .NET implementation**, not integrated into CWA.

You have three options to use it:
- **A) Dual servers:** Run both CWA builtin + kosync-dotnet separately (testing)
- **B) Replace:** Use only kosync-dotnet (loses CWA's book ID/ReadBook/Kobo features)
- **C) Integrate:** Adapt kosync-dotnet to use CWA's databases (best, hardest)

**Recommendation:** Keep CWA's builtin unless kosync-dotnet offers critical features.

---

## 🔗 Related Code Locations

```
Calibre-Web-Automated Repository
├── cps/progress_syncing/
│   ├── __init__.py
│   ├── models.py               # Database models
│   ├── checksums/
│   │   ├── koreader.py        # Partial MD5 algorithm
│   │   └── manager.py         # Checksum storage/retrieval
│   └── protocols/
│       └── kosync.py          # Main KOSync server (750 lines)
│
├── cps/main.py                 # Blueprint registration (line 31)
├── cps/ub.py                   # Database initialization (line 717)
├── README.md                   # Official documentation
│
└── .documentation/             # This folder
    ├── README.md              # This file
    ├── KOSYNC_QUICK_REFERENCE.md
    ├── KOSYNC_ARCHITECTURE.md
    ├── KOSYNC_URLS_CONFIG.md
    ├── KOSYNC_DOTNET_INTEGRATION.md
    └── KOSYNC_DEVELOPMENT.md

extras/
└── kosync-dotnet/            # Submodule: https://github.com/jberlyn/kosync-dotnet
```

---

## 🚀 Getting Started (30 seconds)

### 1. Verify CWA is Running
```bash
curl http://your-instance:8083
```

### 2. Test KOSync Auth
```bash
curl -X GET \
  -H "Authorization: Basic $(echo -n 'admin:admin123' | base64)" \
  http://your-instance:8083/kosync/users/auth
```
Should return: `{"authorized": "OK"}`

### 3. Configure KOReader
- Settings → Synchronization → KOSync
- Server: `http://your-instance:8083`
- Username: [your CWA username]
- Password: [your CWA password]

### 4. Test Sync
- Open a book in KOReader
- Read a few pages
- Trigger sync manually
- Check CWA logs: `docker logs calibre-web-automated | grep kosync`

### 5. Verify in CWA
- Open CWA Admin Panel
- Check CWA Stats page
- `kosync_progress` count should increase
- Book should show as "reading"

---

## 🔍 Common Tasks

### View Sync Progress
```bash
sqlite3 /config/app.db "SELECT user_id, document, percentage, timestamp FROM kosync_progress LIMIT 10;"
```

### View Book Checksums
```bash
sqlite3 /config/metadata.db "SELECT book, format, checksum, created FROM book_format_checksums WHERE book=42;"
```

### Check Reading Status
```bash
sqlite3 /config/cwa.db "SELECT book_id, read_status, times_started_reading FROM ReadBook WHERE user_id=1;"
```

### View Logs
```bash
docker logs -f calibre-web-automated | grep -i kosync
```

### Test an Endpoint
```bash
# Get progress for a document
curl -X GET \
  -H "Authorization: Basic $(echo -n 'admin:admin123' | base64)" \
  http://your-instance:8083/kosync/syncs/progress/b3fb8f4f8448160365087d6ca05c7fa2
```

---

## ❓ FAQ

**Q: Does KOSync work with all e-readers?**
A: Any device running KOReader (Kobo, PocketBook, Boox, Android, etc.)

**Q: Do I need to generate checksums manually?**
A: No! They're generated automatically at import.

**Q: What if a book isn't identified?**
A: Progress still syncs, just without Calibre metadata. This is normal for books not imported via CWA.

**Q: Can I use KOSync offline?**
A: No, device needs network access to CWA instance. But reading continues offline—syncs when connected.

**Q: Does it work with Kobo native sync?**
A: Yes! Progress syncs between KOReader ↔ CWA ↔ Kobo ecosystem.

**Q: Can I replace the builtin with kosync-dotnet?**
A: Yes, but you'll lose automatic book ID and Kobo integration. See KOSYNC_DOTNET_INTEGRATION.md for options.

**Q: How do I debug sync issues?**
A: Check KOSYNC_URLS_CONFIG.md troubleshooting section, view logs, query databases.

---

## 📞 Support

1. **Check logs:** `docker logs calibre-web-automated`
2. **Query databases:** See "Common Tasks" section above
3. **Review docs:** Search relevant documentation file
4. **Test manually:** Use cURL examples from KOSYNC_URLS_CONFIG.md
5. **Check GitHub:** Issues/discussions at https://github.com/crocodilestick/Calibre-Web-Automated

---

## 📖 Document Revision History

- **v1.0** (2025-01-09): Initial comprehensive documentation
  - KOSYNC_QUICK_REFERENCE.md
  - KOSYNC_ARCHITECTURE.md
  - KOSYNC_URLS_CONFIG.md
  - KOSYNC_DOTNET_INTEGRATION.md
  - KOSYNC_DEVELOPMENT.md

---

## 📝 Contributing to Documentation

If you notice:
- Outdated information
- Missing details
- Unclear explanations
- New features not documented

Please update the relevant file and submit a PR!

---

**Last Updated:** January 9, 2025

For the latest code, see: https://github.com/crocodilestick/Calibre-Web-Automated

