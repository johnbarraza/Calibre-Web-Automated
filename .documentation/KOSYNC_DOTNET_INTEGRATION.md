# KOSync Integration Guide: kosync-dotnet

## Overview

This guide explains how to integrate **kosync-dotnet** (located in `extras/kosync-dotnet`) with Calibre-Web Automated.

**Important:** kosync-dotnet is a **completely external project** (https://github.com/jberlyn/kosync-dotnet). It is **not currently integrated into CWA**. The goal is to understand how to integrate it.

---

## 📋 Current State

### CWA: Built-in Python KOSync Server ✅

CWA **has a fully functional KOSync server** built into the codebase:

```
http://your-cwa-instance:8083/kosync/
```

**Status:** ✅ Production-ready  
**Implementation:** `cps/progress_syncing/protocols/kosync.py`  
**Language:** Python + Flask  
**Database:** Uses CWA's existing databases (app.db, metadata.db, cwa.db)

### kosync-dotnet: External .NET Implementation

kosync-dotnet is a **separate, independent project**:

```
Location: extras/kosync-dotnet (git submodule)
GitHub: https://github.com/jberlyn/kosync-dotnet
Language: C# / .NET
Status: External, not integrated into CWA
Database: Independent (if used standalone)
```

**It is NOT:**
- ❌ Built into CWA
- ❌ Automatically deployed with CWA
- ❌ Using CWA's databases
- ❌ Part of the CWA codebase

**It IS:**
- ✅ A separate KOSync server implementation
- ✅ A potential alternative to CWA's builtin
- ✅ Available as a submodule for reference/integration

---

## 🔀 Integration Options for kosync-dotnet

The goal is to integrate kosync-dotnet's .NET implementation with CWA's Python infrastructure.

### Option A: Standalone kosync-dotnet (Parallel Server)

Deploy kosync-dotnet **separately** from CWA on a different port:

**Architecture:**
```
CWA (Python, Port 8083)
└─ /kosync/ ← Builtin Python server

kosync-dotnet (C#/.NET, Port 5000)
└─ /kosync/ ← External server
```

**Setup:**
1. Build kosync-dotnet from `extras/kosync-dotnet`
2. Deploy on port 5000
3. Configure KOReader to use whichever server URL you want

**Advantages:**
- ✅ Minimal changes to CWA
- ✅ Can test kosync-dotnet independently
- ✅ No integration work required
- ✅ Easy to switch between servers

**Disadvantages:**
- ❌ Two separate servers to maintain
- ❌ Two separate databases
- ❌ Data fragmentation (progress split between servers)
- ❌ No integration with CWA's ReadBook/Kobo features

**When to use:** Development/testing, comparing implementations

---

### Option B: Replace CWA's Builtin with kosync-dotnet

Use **only** kosync-dotnet as the KOSync server:

**Architecture:**
```
CWA (Python, Port 8083)
└─ All routes EXCEPT /kosync/

kosync-dotnet (C#/.NET, runs on Port 8083)
└─ /kosync/ ← Handles ALL sync requests
```

**Setup:**
1. Disable CWA's KOSync in `cps/main.py` (comment out blueprint registration)
2. Deploy kosync-dotnet to respond on port 8083
3. Configure in reverse proxy or Docker networking
4. KOReader points to single URL

**Advantages:**
- ✅ Single unified server address
- ✅ Use kosync-dotnet's codebase
- ✅ Only one server to maintain

**Disadvantages:**
- ❌ Lose CWA's integrations:
  - No automatic book identification (unless you add it)
  - No ReadBook status updates
  - No Kobo device sync
  - No `times_started_reading` counter
- ❌ kosync-dotnet needs adaptation:
  - Must support CWA's user authentication
  - Must support RFC 7617 Basic Auth
  - Must support LDAP (if enabled)
- ❌ Data migration required (move existing progress)
- ❌ Ongoing maintenance burden to keep compatible

**When to use:** If kosync-dotnet has critical features CWA's builtin lacks

---

### Option C: Integrate kosync-dotnet into CWA (Full Integration) ⭐ BEST

**Replace** CWA's Python KOSync implementation with kosync-dotnet, **integrated into CWA's ecosystem**:

**Architecture:**
```
CWA (Python, Port 8083)
├─ All CWA routes (web UI, admin, etc.)
│
└─ /kosync/ ← kosync-dotnet (C#/.NET) as integrated module
   ├─ Uses: CWA's app.db, metadata.db, cwa.db
   ├─ Auth: CWA's user system + LDAP fallback
   ├─ Integration: Automatic book ID, ReadBook sync, Kobo sync
   └─ Deployment: Docker container with .NET runtime
```

**Setup Process:**
1. Analyze kosync-dotnet codebase
2. Create adapter/wrapper layer:
   ```
   cps/progress_syncing/protocols/kosync_dotnet_adapter.py
   └─ Flask Blueprint that calls kosync-dotnet APIs
   └─ Maps results to CWA database schemas
   ```
3. OR: Compile/call kosync-dotnet via subprocess/IPC
4. Share databases with CWA
5. Share authentication system
6. Update Docker image to include .NET runtime

**Advantages:**
- ✅ Keep all CWA features (book ID, ReadBook, Kobo)
- ✅ Use kosync-dotnet's implementation
- ✅ Single unified database
- ✅ Single server on port 8083
- ✅ No data fragmentation
- ✅ Better performance integration
- ✅ Seamless for users

**Disadvantages:**
- ⚠️ **Highest complexity** - significant development effort
- ⚠️ Cross-language integration (.NET ↔ Python)
- ⚠️ Need to understand both codebases deeply
- ⚠️ Thorough testing required
- ⚠️ Ongoing maintenance (keep both in sync)
- ⚠️ Docker image needs .NET runtime
- ⚠️ Database schema mapping required

**When to use:** If kosync-dotnet offers significant benefits worth the effort

---

## 🎯 Recommended Approach

### For Initial Testing: **Option A (Standalone)**

```yaml
# docker-compose.yml.dev
services:
  calibre-web-automated:
    ports:
      - "8083:8083"
    # KOSync builtin at http://localhost:8083/kosync/
  
  kosync-dotnet:
    build: ./extras/kosync-dotnet
    ports:
      - "5000:5000"
    # kosync-dotnet at http://localhost:5000/
```

**Pros:**
- ✅ Test kosync-dotnet independently
- ✅ Compare with CWA's implementation
- ✅ No risk to existing CWA functionality
- ✅ Understand kosync-dotnet's capabilities

**Next:** Analyze if integration is worth the effort.

---

### For Production Integration: **Option C (Full Integration)**

If kosync-dotnet offers compelling advantages:

1. **Analyze kosync-dotnet:**
   - What features does it have that CWA's builtin doesn't?
   - What's the performance difference?
   - How well-maintained is it?
   - Can it work with CWA's databases?

2. **Plan integration:**
   - Create adapter layer (Python ↔ .NET bridge)
   - Map kosync-dotnet's database to CWA's schemas
   - Share authentication system
   - Test thoroughly

3. **Implement:**
   - Option C architecture (see above)
   - Adapt Docker image
   - Migrate existing progress data

**Trade-off:** Significant development effort for best UX and features.

---

## 🔌 Adaptation Required for kosync-dotnet Integration

**kosync-dotnet will need modifications** to work with CWA, regardless of integration approach.

### 1. User Authentication

kosync-dotnet must support:
```python
# HTTP Basic Auth (RFC 7617)
Authorization: Basic base64(username:password)

# Validate against CWA's user table (ub.User)
# Support LDAP fallback if enabled
```

**Current kosync-dotnet:** Unknown if it supports this. Must verify/implement.

### 2. Database Access

**For Option A (Standalone):** Independent database (no changes needed)

**For Option B (Replacement):** 
- kosync-dotnet must read from CWA's databases
- app.db → user table for authentication
- metadata.db → books table for titles/paths
- app.db → ReadBook table to update status

**For Option C (Integration):**
- kosync-dotnet (C#) must write to SQLite
- Map kosync-dotnet's data structures to CWA schemas
- Support CWA's SQLAlchemy ORM
- Handle database transactions properly

### 3. Response Format

kosync-dotnet must return:
```json
{
    "document": "partial-md5",
    "progress": "position-string",
    "percentage": 0.4567,                    # decimal 0-1
    "device": "device-name",
    "device_id": "id",
    "timestamp": 1699564800,
    "calibre_book_id": 42,                  # Optional
    "calibre_book_title": "Title",          # Optional
    "calibre_book_format": "EPUB",          # Optional
    "calibre_checksum_version": "koreader"  # Optional
}
```

### 4. Checksum Compatibility

kosync-dotnet must:
- ✅ Use KOReader's partial MD5 algorithm
- ✅ Match CWA's `calculate_koreader_partial_md5()` implementation
- ✅ Store checksums compatible with CWA's schema
- ✅ Look up books by partial MD5

### 5. Reading Status Updates

kosync-dotnet must:
```
0% → UNREAD
1-98% → IN_PROGRESS
  └─ Increment times_started_reading
99-100% → FINISHED
```

And update:
- ReadBook status
- KoboReadingState (if applicable)
- Timestamp of last modification

### 6. API Endpoints

kosync-dotnet must implement:
```
GET /kosync/users/auth              # Authenticate user
GET /kosync/syncs/progress/<doc>    # Get progress
PUT /kosync/syncs/progress          # Update progress
GET /kosync                          # Plugin download (optional)
```

---

## 📊 Comparison: CWA Builtin vs kosync-dotnet

| Feature | CWA Builtin | kosync-dotnet | Status |
|---------|------------|---------------|--------|
| **HTTP Endpoints** | ✅ Complete | ? | **VERIFY** |
| **Basic Auth (RFC 7617)** | ✅ Yes | ? | **VERIFY** |
| **Partial MD5 (KOReader)** | ✅ Implemented | ? | **VERIFY** |
| **Book Identification** | ✅ Auto via checksums | ? | **VERIFY** |
| **ReadBook Status Update** | ✅ Yes | ? | **VERIFY** |
| **KoboReadingState Sync** | ✅ Yes | ? | **VERIFY** |
| **LDAP Support** | ✅ Fallback | ? | **VERIFY** |
| **CWA Database Integration** | ✅ Native | ❌ No | **NEEDS WORK** |
| **User Authentication** | ✅ CWA users | ? | **VERIFY** |
| **Error Handling** | ✅ Standard codes | ? | **VERIFY** |
| **Production Ready** | ✅ Yes | ? | **UNKNOWN** |
| **Maintenance** | ✅ Active (CWA team) | ? | **UNKNOWN** |

**Key Question:** What does kosync-dotnet do **better** than CWA's builtin that justifies the integration effort?

---

## 🚀 Deployment Decision Tree

```
Do you want to use kosync-dotnet?
│
├─ NO → Use CWA builtin (current, no changes needed)
│
└─ YES
   │
   ├─ Want to test it first?
   │  └─ YES → Option A (Dual servers on different ports)
   │
   └─ Want production integration?
      │
      ├─ Simple replacement (lose CWA features)?
      │  └─ YES → Option B (Replace builtin)
      │
      └─ Keep all CWA features?
         └─ YES → Option C (Integrate as wrapper)
```

---

## � Deployment Decision Tree

```
Goal: Integrate kosync-dotnet with CWA?
│
├─ NO → Use CWA's builtin KOSync (no action needed)
│
└─ YES → What's the goal?
   │
   ├─ Test/Compare kosync-dotnet
   │  └─ Option A: Run standalone on port 5000
   │     └─ Build from extras/kosync-dotnet
   │     └─ Configure KOReader to use whichever you want
   │     └─ Compare implementations
   │     └─ Decide if worth integrating
   │
   └─ kosync-dotnet has features we need
      │
      ├─ Want quick deployment (lose CWA features)?
      │  └─ Option B: Replace builtin
      │     └─ Disable CWA KOSync in cps/main.py
      │     └─ Deploy kosync-dotnet on port 8083
      │     └─ Adapt kosync-dotnet for CWA auth
      │     └─ ⚠️ Lose auto book ID, ReadBook sync, Kobo sync
      │
      └─ Want full integration (keep CWA features)?
         └─ Option C: Integrate into CWA
            ├─ Analyze kosync-dotnet codebase thoroughly
            ├─ Create Python adapter layer
            ├─ Share CWA databases (app.db, metadata.db, cwa.db)
            ├─ Share authentication system
            ├─ Update Docker image with .NET runtime
            ├─ ✅ Keep all CWA features
            ├─ ✅ Single unified server
            └─ ⚠️ Significant development effort
```

---

## 📚 Reference Files

**Builtin KOSync:**
- Main: `cps/progress_syncing/protocols/kosync.py` (750 lines)
- Models: `cps/progress_syncing/models.py`
- Checksums: `cps/progress_syncing/checksums/koreader.py`
- Registration: `cps/main.py` line 31

**Integration Points:**
- Database: `cps/ub.py`, `cps/db.py`
- User Auth: `cps/MyLoginManager.py`
- Tasks: `cps/services/worker.py`

---

## ⚠️ Important Considerations

1. **Data Migration:** If switching, plan how to migrate existing progress data
2. **User Communication:** If changing endpoints, inform users
3. **Backward Compatibility:** Builtin will continue working unless explicitly disabled
4. **Testing:** Thoroughly test with actual KOReader devices before production
5. **Database Locks:** Monitor SQLite locks during concurrent syncs

---

## 🎯 Next Steps

1. **Evaluate kosync-dotnet:** What advantages does it offer over builtin?
2. **Choose strategy:** A, B, or C based on your goals
3. **Prototype:** Start with Option A (dual servers) for low-risk testing
4. **Plan migration:** If moving to B or C, plan data migration
5. **Test thoroughly:** Use actual KOReader devices

---

## 💬 Questions to Answer

- Does kosync-dotnet support the same API as CWA builtin?
- Does it use the same partial MD5 algorithm?
- Can it write to CWA's database schemas?
- What authentication methods does it support?
- Is it more performant or stable than the builtin?
- Does it support LDAP?

