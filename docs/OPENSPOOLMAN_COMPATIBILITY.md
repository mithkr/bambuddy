# OpenSpoolMan Compatibility Layer - Technical Documentation

**Document Version:** 1.0  
**Generated:** 2026-05-14  
**Target Repository:** OpenSpoolMan  
**Source Analysis:** maziggy/bambuddy @ a108394d805381bcf35847d9b12cf514a577c9a5

---

## Table of Contents

1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Slicer Filament Identifiers](#slicer-filament-identifiers)
4. [Data Storage Locations](#data-storage-locations)
5. [API Endpoint Contract](#api-endpoint-contract)
6. [Database Schema](#database-schema)
7. [Key Data Flows](#key-data-flows)
8. [Implementation Checklist](#implementation-checklist)
9. [Critical Implementation Notes](#critical-implementation-notes)
10. [Example Lifecycle](#example-lifecycle)
11. [Testing Strategy](#testing-strategy)
12. [References](#references)

---

## Overview

Bambuddy integrates with Spoolman to provide centralized filament inventory management across multiple Bambu Lab 3D printers. This document details the integration contract that OpenSpoolMan must implement to achieve full compatibility with Bambuddy.

### What is OpenSpoolMan Compatibility?

OpenSpoolMan compatibility means implementing the **exact same HTTP API contract** that Bambuddy uses to interact with Spoolman, allowing Bambuddy to work seamlessly with OpenSpoolMan as a drop-in replacement.

### Key Principles

- **No Breaking Changes**: OpenSpoolMan must implement 100% of Spoolman's public API endpoints used by Bambuddy
- **Extra Fields Are Critical**: Custom fields (`bambu_slicer_filament`, `bambu_slicer_filament_name`, `tag`) are stored as JSON-encoded strings in the `extra` dict
- **Local Slot Tracking**: Bambuddy maintains its own database table (`spoolman_slot_assignments`) for AMS slot assignments—NOT stored in OpenSpoolMan
- **Two ID Formats**: Understanding the difference between `setting_id` (GFSL05) and `filament_id` (GFL05) is crucial

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Bambuddy Frontend                      │
│              (React + TypeScript + TanStack Query)          │
└──────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    Bambuddy Backend APIs                    │
│  (/api/v1/spoolman/*, /api/v1/inventory/*, etc.)           │
└──────────────────────────────────────────────────────────────┘
                              ↓
            ┌─────────────────────────────────────────┐
            │   HTTP REST (Spoolman API Contract)    │
            └─────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────────┐
    │          Spoolman (or OpenSpoolMan)                │
    │  • Filament catalog (material, brand, color, etc.)  │
    │  • Spool inventory (per-spool tracking)             │
    │  • Extra fields (custom metadata)                   │
    └──────────────────────────────────────────────────────┘
                              ↓
    ┌──────────────────────────────────────────────────────┐
    │      Bambuddy Local Database (PostgreSQL/SQLite)    │
    │  • spoolman_slot_assignments (AMS → spool mapping)  │
    │  • spoolman_k_profile (K-value calibrations)        │
    │  • Other inventory & printer state                  │
    └──────────────────────────────────────────────────────┘
```

### Data Flow Diagram

```
User creates spool with slicer preset
         ↓
Frontend sends: { slicer_filament: "GFSL05", slicer_filament_name: "..." }
         ↓
Backend converts: GFSL05 (setting_id) → stores as JSON in extra field
         ↓
Spoolman stores: extra.bambu_slicer_filament = '"GFSL05"' (JSON-encoded)
         ↓
Later: User assigns spool to AMS slot
         ↓
Backend: Fetches spool from Spoolman, reads extra field, unwraps JSON
         ↓
Conversion: GFSL05 (setting_id) → GFL05 (filament_id) via setting_id_to_filament_id()
         ↓
Backend: Stores (printer_id, ams_id, tray_id, spoolman_spool_id) in LOCAL DB
         ↓
MQTT: Sends ams_set_filament_setting with tray_info_idx="GFL05" to printer
```

---

## Slicer Filament Identifiers

### Two ID Formats - Critical Understanding

Bambu Lab uses **two different identifier formats** for slicer presets depending on context:

#### 1. **Setting ID** (Used by Cloud API & BambuStudio)
- **Format**: `GFSL05`, `GFSG02`, `GFSA00` or `PFUSxxxxxxxx...` (cloud user presets)
- **Prefix Convention**:
  - `GFS` = Official Bambu filament (setting format)
  - `PFUS` = User cloud filament preset
- **Version suffix**: `GFSL05_07` (version 7)
- **Used by**: BambuStudio UI, Bambu Cloud preset API, Bambuddy's spool form
- **Example**: `GFSL05_09` = Bambu PLA Basic v9

#### 2. **Filament ID** (Used by Printer Firmware)
- **Format**: `GFL05`, `GFG02`, `GFA00` or `PFUSxxxxxxxx...` (cloud user presets)
- **Conversion rule**: Remove the `S` after `GF` (i.e., `GFSL05` → `GFL05`)
- **Prefix Convention**:
  - `GF` = Official Bambu filament (filament format, no S)
  - `PFUS` = Same hash as setting_id (no conversion needed)
- **Used by**: Printer firmware calibration tables, MQTT `ams_set_filament_setting`
- **Why it matters**: Printer's calibration DB is keyed by filament_id, not setting_id

### Conversion Functions

```python
def setting_id_to_filament_id(setting_id: str) -> str:
    """GFSL05 → GFL05, PFUS... → PFUS..., etc."""
    if setting_id.startswith("GFS"):
        return f"GF{setting_id[3:]}"  # Remove the 'S'
    return setting_id  # User presets unchanged

def filament_id_to_setting_id(filament_id: str) -> str:
    """GFL05 → GFSL05, PFUS... → PFUS..., etc."""
    if filament_id.startswith("GF") and filament_id[2] != "S":
        return f"GFS{filament_id[2:]}"  # Insert an 'S'
    return filament_id  # User presets unchanged

def normalize_slicer_filament(value: str) -> tuple[str, str]:
    """Returns (tray_info_idx, setting_id) both in base form (no version suffix)."""
    base = value.split("_")[0] if "_" in value else value  # Strip version
    return (setting_id_to_filament_id(base), filament_id_to_setting_id(base))
```

### Generic Fallback IDs

When a spool has no preset selected, Bambuddy maps by material type:

| Material | Filament ID | Setting ID | Used When |
|----------|-------------|-----------|-----------|
| PLA | `GFL99` | `GFSL99` | Generic PLA fallback |
| PETG | `GFG99` | `GFSG99` | Generic PETG fallback |
| ABS | `GFB99` | `GFSB99` | Generic ABS fallback |
| PETG HF | `GFG96` | `GFSG96` | High-flow PETG |
| PLA-CF | `GFL98` | `GFSL98` | Carbon-filled PLA |

---

## Data Storage Locations

### 1. Slicer Preset ID (stored in Spoolman)

**Location**: Spoolman spool record → `extra` dict → JSON-encoded strings

**Fields**:
- `extra.bambu_slicer_filament`: The selected preset ID (e.g., `"GFSL05"` as JSON string `'"GFSL05"'`)
- `extra.bambu_slicer_filament_name`: Human-readable preset name (e.g., `"Bambu PLA Basic @BBL X1C"`)
- `extra.tag`: RFID tag UID or tray UUID (e.g., `"A1B2C3D4E5F6G7H8"`)

**Example Raw Spoolman Spool Object**:
```json
{
  "id": 42,
  "filament": {
    "id": 5,
    "name": "PLA Basic",
    "material": "PLA",
    "color_hex": "FF0000",
    "weight": 1000,
    "vendor": { "name": "Bambu Lab" }
  },
  "remaining_weight": 800.0,
  "extra": {
    "bambu_slicer_filament": "\"GFSL05\"",
    "bambu_slicer_filament_name": "\"Bambu PLA Basic @BBL X1C\"",
    "tag": "\"A1B2C3D4E5F6G7H8\""
  }
}
```

**Critical**: All extra field values are **JSON-encoded strings**:
- User sees: `GFSL05`
- Stored in Spoolman: `'"GFSL05"'` (six characters including quotes)
- Must be unwrapped: `json.loads('"GFSL05"')` → `'GFSL05'`

### 2. AMS Slot Assignment (stored in Bambuddy Local DB)

**Table**: `spoolman_slot_assignments` (only in Bambuddy, NOT in Spoolman)

**Schema**:
```sql
CREATE TABLE spoolman_slot_assignments (
    id INTEGER PRIMARY KEY,
    printer_id INTEGER NOT NULL,        -- Foreign key to printers
    ams_id INTEGER NOT NULL,            -- AMS unit: 0-7, or 255 (external)
    tray_id INTEGER NOT NULL,           -- Tray slot: 0-3
    spoolman_spool_id INTEGER NOT NULL, -- Reference to Spoolman spool
    assigned_at DATETIME DEFAULT NOW(),
    UNIQUE(printer_id, ams_id, tray_id) -- One spool per slot
);
```

**Why Local?** Spoolman's `spool.location` field is user-editable and used for warehouse location (shelf, bin, etc.), not for AMS slot assignments. Bambuddy maintains its own tracking to avoid collision.

### 3. K-Profile (Calibration) Data (stored in Bambuddy Local DB)

**Table**: `spoolman_k_profile`

**Schema**:
```sql
CREATE TABLE spoolman_k_profile (
    id INTEGER PRIMARY KEY,
    spoolman_spool_id INTEGER NOT NULL,
    printer_id INTEGER NOT NULL,
    extruder INTEGER DEFAULT 0,          -- 0=right, 1=left (dual nozzle)
    nozzle_diameter VARCHAR(10) DEFAULT '0.4',
    k_value REAL NOT NULL,               -- K-value for flow dynamics
    name VARCHAR(100),                   -- Profile name
    cali_idx INTEGER,                    -- Calibration index on printer
    setting_id VARCHAR(50),              -- Linked filament preset ID
    created_at DATETIME DEFAULT NOW(),
    UNIQUE(spoolman_spool_id, printer_id, extruder, nozzle_diameter)
);
```

---

## API Endpoint Contract

### Required Spoolman Endpoints (for OpenSpoolMan)

#### Health & Status

```http
GET /api/v1/health
```
**Response:**
```json
{
  "version": "1.2.3"
}
```

#### Filament Management

```http
POST /api/v1/filament
Content-Type: application/json

{
  "name": "Devil Design PLA Basic",
  "material": "PLA",
  "vendor_id": 5,
  "color_hex": "FF0000",
  "weight": 1000,
  "settings_extruder_temp": 210
}
```

```http
GET /api/v1/filament
```

```http
GET /api/v1/filament/{id}
```

```http
PATCH /api/v1/filament/{id}
```

#### Spool Management

```http
POST /api/v1/spool
Content-Type: application/json

{
  "filament_id": 5,
  "remaining_weight": 800.0,
  "location": "Shelf A1",
  "lot_nr": "LOT123",
  "comment": "Purchased 2024-01-01",
  "extra": {}
}
```

```http
GET /api/v1/spool
```

```http
GET /api/v1/spool/{id}
```

```http
PATCH /api/v1/spool/{id}
Content-Type: application/json

{
  "remaining_weight": 750.0,
  "extra": { "bambu_slicer_filament": "\"GFSL05\"" }
}
```

#### Extra Field Management (CRITICAL)

```http
POST /api/v1/extra-field
Content-Type: application/json

{
  "field_name": "bambu_slicer_filament"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Field registered"
}
```

**Why this is critical**: Spoolman validates extra field names before allowing writes. OpenSpoolMan must support:
- `bambu_slicer_filament` - Preset ID
- `bambu_slicer_filament_name` - Preset display name
- `tag` - RFID/UUID tag (existing in Spoolman)

If these fields are not registered, PATCH operations with unknown extra keys will return **HTTP 400/502**.

#### Vendor Management

```http
POST /api/v1/vendor
Content-Type: application/json

{
  "name": "Devil Design"
}
```

```http
GET /api/v1/vendor
```

---

## Database Schema

### Spoolman Core Tables (OpenSpoolMan must implement)

```sql
-- Vendors (brand/manufacturer)
CREATE TABLE vendor (
    id INTEGER PRIMARY KEY,
    name VARCHAR(128) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Filaments (material types)
CREATE TABLE filament (
    id INTEGER PRIMARY KEY,
    name VARCHAR(128) NOT NULL,
    material VARCHAR(64) NOT NULL,
    color_hex VARCHAR(6),
    color_name VARCHAR(64),
    weight INTEGER DEFAULT 1000,
    spool_weight INTEGER DEFAULT 250,
    density REAL,
    vendor_id INTEGER FOREIGN KEY,
    settings_extruder_temp INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Spools (individual spools in inventory)
CREATE TABLE spool (
    id INTEGER PRIMARY KEY,
    filament_id INTEGER NOT NULL FOREIGN KEY,
    remaining_weight REAL DEFAULT 1000,
    used_weight REAL DEFAULT 0,
    location VARCHAR(255),
    lot_nr VARCHAR(64),
    comment TEXT,
    price REAL,
    first_used TIMESTAMP,
    last_used TIMESTAMP,
    registered TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    archived BOOLEAN DEFAULT FALSE,
    extra TEXT DEFAULT '{}'  -- JSON object for Bambuddy extra fields
);

-- Extra Field Registry (metadata about custom fields)
CREATE TABLE extra_field (
    id INTEGER PRIMARY KEY,
    field_name VARCHAR(128) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Bambuddy Local Tables (NOT in Spoolman)

```sql
-- Printer AMS slot → Spoolman spool mapping
CREATE TABLE spoolman_slot_assignments (
    id INTEGER PRIMARY KEY,
    printer_id INTEGER NOT NULL FOREIGN KEY,
    ams_id INTEGER NOT NULL CHECK ((ams_id >= 0 AND ams_id <= 7) OR ams_id = 255),
    tray_id INTEGER NOT NULL CHECK (tray_id >= 0 AND tray_id <= 3),
    spoolman_spool_id INTEGER NOT NULL,
    assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(printer_id, ams_id, tray_id)
);

-- K-value calibration profiles linked to spools
CREATE TABLE spoolman_k_profile (
    id INTEGER PRIMARY KEY,
    spoolman_spool_id INTEGER NOT NULL,
    printer_id INTEGER NOT NULL FOREIGN KEY,
    extruder INTEGER DEFAULT 0 CHECK (extruder >= 0 AND extruder <= 1),
    nozzle_diameter VARCHAR(10) DEFAULT '0.4',
    k_value REAL NOT NULL,
    name VARCHAR(100),
    cali_idx INTEGER,
    setting_id VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(spoolman_spool_id, printer_id, extruder, nozzle_diameter)
);
```

---

## Key Data Flows

### Flow 1: Creating a Spool with Slicer Preset

```
User Action: Create spool form with preset selection
             ↓
Frontend sends:
  POST /api/v1/spoolman/inventory/spools
  {
    "material": "PLA",
    "brand": "Devil Design",
    "color_hex": "FF0000",
    "label_weight": 1000,
    "slicer_filament": "GFSL05",           ← setting_id
    "slicer_filament_name": "Bambu PLA..."
  }
             ↓
Backend:
  1. Lookup/create vendor, filament in Spoolman
  2. Ensure extra fields registered:
     - POST /api/v1/extra-field { "field_name": "bambu_slicer_filament" }
     - POST /api/v1/extra-field { "field_name": "bambu_slicer_filament_name" }
  3. Create spool with extra fields:
     POST /api/v1/spool {
       "filament_id": <spoolman_filament_id>,
       "remaining_weight": 1000,
       "extra": {
         "bambu_slicer_filament": "\"GFSL05\"",       ← JSON-encoded
         "bambu_slicer_filament_name": "\"Bambu...\""
       }
     }
             ↓
Spoolman Response:
  {
    "id": 42,
    "filament": { "id": 5, "name": "PLA Basic", ... },
    "remaining_weight": 1000,
    "extra": {
      "bambu_slicer_filament": "\"GFSL05\"",
      "bambu_slicer_filament_name": "\"Bambu...\""
    }
  }
             ↓
Frontend receives mapped spool object
```

### Flow 2: Assigning Spool to AMS Slot

```
User Action: Click "Assign to AMS 0 Tray 1"
             ↓
Frontend sends:
  POST /api/v1/spoolman/inventory/slot-assignments
  {
    "spoolman_spool_id": 42,
    "printer_id": 1,
    "ams_id": 0,
    "tray_id": 1
  }
             ↓
Backend:
  1. Fetch spool from Spoolman:
     GET /api/v1/spool/42
  2. Extract and unwrap extra fields:
     - Read: extra.bambu_slicer_filament = "\"GFSL05\""
     - Parse: json.loads("\"GFSL05\"") → "GFSL05"
  3. Convert setting_id to filament_id:
     - "GFSL05" → "GFL05" (tray_info_idx for MQTT)
  4. Store local assignment:
     INSERT INTO spoolman_slot_assignments 
       (printer_id, ams_id, tray_id, spoolman_spool_id) 
       VALUES (1, 0, 1, 42)
  5. Send MQTT command to printer:
     ams_set_filament_setting(
       ams_id=0,
       tray_id=1,
       tray_info_idx="GFL05",           ← filament_id
       tray_type="PLA",
       tray_sub_brands="Bambu PLA Basic",
       tray_color="FF0000FF",
       setting_id="GFSL05"              ← setting_id for slicer
     )
             ↓
Printer: Configures slot with K-value, temperature, etc.
         Bambuddy stores assignment in local DB
```

### Flow 3: Syncing Printer AMS to Spoolman (RFID Tag)

```
Printer State: AMS has RFID-tagged spool
               tray_tag_uid = "A1B2C3D4E5F6G7H8"
               tray_info_idx = "GFL05"
             ↓
Backend calls:
  POST /api/v1/spoolman/sync/{printer_id}
             ↓
Backend Process:
  1. Read printer AMS data via MQTT
  2. For each tray with RFID:
     a. Search Spoolman spools for matching tag:
        GET /api/v1/spool?extra.tag=...
     b. If found: Update slot assignment
        INSERT INTO spoolman_slot_assignments 
          (printer_id, ams_id, tray_id, spoolman_spool_id)
          VALUES (1, 0, 1, <spool_id>)
  3. If NOT found but has local slot assignment:
     Use hint to maintain assignment without RFID
             ↓
Response:
  {
    "success": true,
    "synced_count": 3,
    "skipped": []
  }
```

### Flow 4: Updating Spool (e.g., Change Preset)

```
User Action: Edit spool, change preset from GFL05 to GFG02
             ↓
Frontend sends:
  PATCH /api/v1/spoolman/inventory/spools/42
  {
    "slicer_filament": "GFSG02",
    "slicer_filament_name": "Bambu PETG HF"
  }
             ↓
Backend:
  1. Ensure extra fields registered
  2. Prepare extra dict:
     {
       "bambu_slicer_filament": "\"GFSG02\"",
       "bambu_slicer_filament_name": "\"Bambu PETG HF\""
     }
  3. PATCH spool:
     PATCH /api/v1/spool/42
     {
       "extra": { ... }  ← Spoolman MERGES this with existing
     }
             ↓
Spoolman: Merges new extra fields with existing
          (does NOT overwrite, only updates specified keys)
```

---

## Implementation Checklist

### Phase 1: Foundation (Weeks 1-2)

- [ ] Fork OpenSpoolMan repository (or create new one)
- [ ] Set up Python/FastAPI project structure
- [ ] Implement core database models:
  - [ ] `vendor` table
  - [ ] `filament` table
  - [ ] `spool` table
  - [ ] `extra_field` registry table
- [ ] Create SQLAlchemy ORM models
- [ ] Set up async database session management
- [ ] Implement basic authentication/authorization (if needed)

### Phase 2: Core Spoolman API Endpoints (Weeks 3-4)

- [ ] Health check endpoint (`GET /api/v1/health`)
- [ ] Vendor CRUD endpoints:
  - [ ] `POST /api/v1/vendor`
  - [ ] `GET /api/v1/vendor`
  - [ ] `GET /api/v1/vendor/{id}`
- [ ] Filament CRUD endpoints:
  - [ ] `POST /api/v1/filament`
  - [ ] `GET /api/v1/filament`
  - [ ] `GET /api/v1/filament/{id}`
  - [ ] `PATCH /api/v1/filament/{id}`
- [ ] Spool CRUD endpoints:
  - [ ] `POST /api/v1/spool`
  - [ ] `GET /api/v1/spool`
  - [ ] `GET /api/v1/spool/{id}`
  - [ ] `PATCH /api/v1/spool/{id}` (with extra dict MERGE logic)
- [ ] Extra field management:
  - [ ] `POST /api/v1/extra-field` (register field names)
  - [ ] Validation: reject unknown extra keys on write

### Phase 3: Bambuddy Integration Features (Weeks 5-6)

- [ ] Implement extra field handling:
  - [ ] JSON encoding/decoding for extra dict values
  - [ ] Field name validation before accepting writes
  - [ ] Ensure `tag`, `bambu_slicer_filament`, `bambu_slicer_filament_name` fields
- [ ] Spool extra field MERGE semantics:
  - [ ] `PATCH /api/v1/spool/{id}` merges extra dict, not replaces
  - [ ] Validate all extra keys are registered before accepting
- [ ] Implement filter/search:
  - [ ] `GET /api/v1/spool?material=PLA`
  - [ ] `GET /api/v1/spool?extra.tag=...`

### Phase 4: Testing & Documentation (Weeks 7-8)

- [ ] Unit tests for all endpoints
- [ ] Integration tests with Bambuddy API contract
- [ ] Documentation:
  - [ ] API OpenAPI/Swagger spec
  - [ ] Database schema documentation
  - [ ] Installation & setup guide
- [ ] Compatibility matrix (Spoolman v1.x, v2.x)

### Phase 5: Deployment & Iteration (Weeks 9+)

- [ ] Docker Compose configuration (matching Spoolman's pattern)
- [ ] Migration guide (Spoolman → OpenSpoolMan data import)
- [ ] Performance optimization
- [ ] Community feedback & refinement

---

## Critical Implementation Notes

### 1. **JSON Encoding of Extra Fields**

**Problem**: Extra dict values in Spoolman are always stored as **JSON-encoded strings**, not raw values.

**Requirement for OpenSpoolMan**:

```python
# When WRITING to extra dict (from Bambuddy):
import json
extra_data = {
    "bambu_slicer_filament": json.dumps("GFSL05"),  # → '"GFSL05"' (6 chars)
    "bambu_slicer_filament_name": json.dumps("Bambu PLA Basic"),
    "tag": json.dumps("A1B2C3D4E5F6G7H8")
}
# Store `extra_data` as-is (already JSON-encoded)

# When READING from extra dict (to Bambuddy):
stored_extra = {
    "bambu_slicer_filament": '"GFSL05"',  # As stored
    "tag": '"A1B2C3D4E5F6G7H8"'
}
# Must UNWRAP:
def extract_extra_str(extra: dict, key: str) -> str:
    raw = extra.get(key, "")
    if isinstance(raw, str):
        try:
            return json.loads(raw)  # Unwrap JSON encoding
        except (json.JSONDecodeError, ValueError):
            return raw  # Fallback to bare string
    return ""

slicer_filament = extract_extra_str(stored_extra, "bambu_slicer_filament")
# → "GFSL05" (unwrapped)
```

### 2. **Extra Field Registration Before Write**

**Problem**: Spoolman rejects `PATCH /api/v1/spool/{id}` with unknown extra keys.

**Requirement for OpenSpoolMan**:

```python
@app.post("/api/v1/extra-field")
async def register_extra_field(field_name: str):
    # Check if field already exists
    existing = await db.query(ExtraField).filter_by(name=field_name).first()
    if existing:
        return {"success": True}
    
    # Register new field
    new_field = ExtraField(name=field_name)
    await db.add(new_field)
    await db.commit()
    
    # Log for monitoring
    logger.info(f"Registered extra field: {field_name}")
    return {"success": True}

# Validation on PATCH:
@app.patch("/api/v1/spool/{spool_id}")
async def update_spool(spool_id: int, data: dict):
    if "extra" in data:
        # Check all extra keys are registered
        registered_fields = await db.query(ExtraField).all()
        allowed_keys = {f.name for f in registered_fields}
        
        for key in data["extra"].keys():
            if key not in allowed_keys:
                raise HTTPException(
                    status_code=400,
                    detail=f"Unknown extra field: {key}"
                )
    
    # ... proceed with update
```

### 3. **Extra Dict MERGE Semantics (Not Replace)**

**Problem**: Spoolman's `PATCH` operation **merges** extra dict keys, it doesn't replace the entire dict.

**Requirement for OpenSpoolMan**:

```python
# User PATCHES with partial extra update:
PATCH /api/v1/spool/42
{
  "extra": { "bambu_slicer_filament": '"GFSG02"' }
}

# Current spool has:
spool.extra = {
  "bambu_slicer_filament": '"GFSL05"',
  "bambu_slicer_filament_name": '"Bambu PLA Basic"',
  "tag": '"A1B2C3D4E5F6G7H8"'
}

# After MERGE (NOT replace):
spool.extra = {
  "bambu_slicer_filament": '"GFSG02"',  ← UPDATED
  "bambu_slicer_filament_name": '"Bambu PLA Basic"',  ← KEPT
  "tag": '"A1B2C3D4E5F6G7H8"'  ← KEPT
}

# WRONG approach (replacement):
# spool.extra = { "bambu_slicer_filament": '"GFSG02"' }  ❌ Loses other fields!

# CORRECT implementation:
async def update_spool(spool_id: int, updates: dict):
    spool = await db.query(Spool).get(spool_id)
    
    if "extra" in updates:
        # MERGE new extra with existing
        spool.extra = {**spool.extra, **updates["extra"]}
    
    # Update other fields
    for key, value in updates.items():
        if key != "extra":
            setattr(spool, key, value)
    
    await db.commit()
```

### 4. **Unique Constraint on (printer_id, ams_id, tray_id)**

**In Bambuddy's local DB** only (not Spoolman), the `spoolman_slot_assignments` table must enforce:

```sql
UNIQUE(printer_id, ams_id, tray_id)
```

This ensures **one spool per slot**. If a user reassigns a slot, an UPSERT is used:

```python
# Insert or update
INSERT INTO spoolman_slot_assignments
  (printer_id, ams_id, tray_id, spoolman_spool_id)
  VALUES (?, ?, ?, ?)
ON CONFLICT(printer_id, ams_id, tray_id)
  DO UPDATE SET spoolman_spool_id = excluded.spoolman_spool_id
```

### 5. **Version Suffix Stripping**

**Problem**: Cloud presets often have version suffixes like `GFSL05_07`.

**Requirement for OpenSpoolMan**: When displaying or converting IDs, always strip the version suffix:

```python
def normalize_filament_id(raw: str) -> str:
    """Strip version suffix: GFSL05_07 → GFSL05"""
    return raw.split("_")[0] if "_" in raw else raw

# Usage:
stored_id = "GFSL05_07"
base_id = normalize_filament_id(stored_id)  # → "GFSL05"
filament_id = setting_id_to_filament_id(base_id)  # → "GFL05"
```

### 6. **NULL vs Empty String in Extra Fields**

**Requirement**: When clearing an extra field, send `json.dumps("")` (the JSON-encoded empty string), not `NULL`:

```python
# CORRECT way to clear a field:
PATCH /api/v1/spool/42
{
  "extra": { "bambu_slicer_filament": '""' }  # JSON-encoded empty string
}

# NOT NULL (Spoolman doesn't clear NULL values in MERGE)
{
  "extra": { "bambu_slicer_filament": null }  # ❌ Wrong
}

# After this PATCH, reading the field returns empty string:
spool.extra["bambu_slicer_filament"]  # → '""'
# Which unwraps to:
json.loads('""')  # → ""
```

---

## Example Lifecycle

### Scenario: Create spool with preset, assign to printer, sync RFID

#### Step 1: User creates spool (Spoolman side)

```bash
# Register extra fields
curl -X POST http://openspoolman:7912/api/v1/extra-field \
  -H "Content-Type: application/json" \
  -d '{"field_name": "bambu_slicer_filament"}'

curl -X POST http://openspoolman:7912/api/v1/extra-field \
  -H "Content-Type: application/json" \
  -d '{"field_name": "bambu_slicer_filament_name"}'

# Create vendor and filament (if not exists)
curl -X POST http://openspoolman:7912/api/v1/vendor \
  -H "Content-Type: application/json" \
  -d '{"name": "Devil Design"}'

curl -X POST http://openspoolman:7912/api/v1/filament \
  -H "Content-Type: application/json" \
  -d '{
    "vendor_id": 1,
    "name": "PLA Basic",
    "material": "PLA",
    "color_hex": "FF0000",
    "weight": 1000,
    "settings_extruder_temp": 210
  }'

# Create spool WITH slicer preset
curl -X POST http://openspoolman:7912/api/v1/spool \
  -H "Content-Type: application/json" \
  -d '{
    "filament_id": 1,
    "remaining_weight": 1000,
    "extra": {
      "bambu_slicer_filament": "\"GFSL05\"",
      "bambu_slicer_filament_name": "\"Bambu PLA Basic @BBL X1C\""
    }
  }'

# Response:
{
  "id": 42,
  "filament": { "id": 1, "name": "PLA Basic", ... },
  "remaining_weight": 1000,
  "extra": {
    "bambu_slicer_filament": "\"GFSL05\"",
    "bambu_slicer_filament_name": "\"Bambu PLA Basic @BBL X1C\""
  }
}
```

#### Step 2: User assigns spool to AMS slot (Bambuddy side)

```python
# Bambuddy backend:

# 1. Fetch spool from OpenSpoolMan
spool = await http_client.get("http://openspoolman:7912/api/v1/spool/42")
# spool.extra["bambu_slicer_filament"] = "\"GFSL05\""

# 2. Unwrap and convert
import json
stored_json = spool["extra"]["bambu_slicer_filament"]
setting_id = json.loads(stored_json)  # → "GFSL05"
filament_id = setting_id_to_filament_id(setting_id)  # → "GFL05"

# 3. Store slot assignment in LOCAL DB
INSERT INTO spoolman_slot_assignments
  (printer_id, ams_id, tray_id, spoolman_spool_id)
  VALUES (1, 0, 1, 42)

# 4. Send MQTT to printer
printer_client.ams_set_filament_setting(
  ams_id=0,
  tray_id=1,
  tray_info_idx="GFL05",  # ← filament_id
  tray_type="PLA",
  tray_sub_brands="Devil Design PLA",
  tray_color="FF0000FF",
  setting_id="GFSL05"  # ← setting_id for slicer
)
```

#### Step 3: Printer reports RFID tag (sync flow)

```python
# Printer MQTT message includes:
{
  "ams": {
    "ams": [
      {
        "id": 0,
        "tray": [
          {
            "id": 1,
            "tag_uid": "A1B2C3D4E5F6G7H8",  # ← RFID detected
            "tray_type": "PLA",
            "tray_info_idx": "GFL05"
          }
        ]
      }
    ]
  }
}

# Bambuddy sync process:
# 1. Fetch all spools from OpenSpoolMan
all_spools = await http_client.get("http://openspoolman:7912/api/v1/spool")

# 2. Search for spool with matching tag
for spool in all_spools:
    stored_tag_json = spool.get("extra", {}).get("tag", "")
    stored_tag = json.loads(stored_tag_json) if stored_tag_json else ""
    if stored_tag == "A1B2C3D4E5F6G7H8":
        # Found! Update or create slot assignment
        INSERT INTO spoolman_slot_assignments
          (printer_id, ams_id, tray_id, spoolman_spool_id)
          VALUES (1, 0, 1, spool["id"])
          ON CONFLICT DO UPDATE ...
        break
```

---

## Testing Strategy

### Unit Tests (API Contract Validation)

```python
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_extra_field_registration():
    """Register custom extra field"""
    client = AsyncClient(base_url="http://localhost:7912/api/v1")
    response = await client.post("/extra-field", json={"field_name": "test_field"})
    assert response.status_code == 200

@pytest.mark.asyncio
async def test_spool_creation_with_extra():
    """Create spool with JSON-encoded extra fields"""
    response = await client.post("/spool", json={
        "filament_id": 1,
        "remaining_weight": 1000,
        "extra": {
            "bambu_slicer_filament": '"GFSL05"',
            "tag": '"A1B2C3D4"'
        }
    })
    assert response.status_code == 201
    spool = response.json()
    assert spool["extra"]["bambu_slicer_filament"] == '"GFSL05"'

@pytest.mark.asyncio
async def test_spool_extra_merge():
    """PATCH merges extra dict, doesn't replace"""
    # Create spool with two fields
    spool_id = 42
    spool = {
        "extra": {
            "field_a": '"value_a"',
            "field_b": '"value_b"'
        }
    }
    
    # Patch with one field
    response = await client.patch(f"/spool/{spool_id}", json={
        "extra": {"field_a": '"updated_a"'}
    })
    
    # Both fields should exist, one updated
    result = response.json()
    assert result["extra"]["field_a"] == '"updated_a"'
    assert result["extra"]["field_b"] == '"value_b"'  # ← Preserved!
```

### Integration Tests (Bambuddy Compatibility)

```python
@pytest.mark.asyncio
async def test_bambuddy_spool_assignment_flow():
    """Full lifecycle: create spool → assign to AMS → verify"""
    
    # 1. Register fields
    await client.post("/extra-field", json={"field_name": "bambu_slicer_filament"})
    
    # 2. Create spool
    spool_response = await client.post("/spool", json={
        "filament_id": 1,
        "remaining_weight": 1000,
        "extra": {
            "bambu_slicer_filament": '"GFSL05"',
            "tag": '"A1B2C3D4E5F6G7H8"'
        }
    })
    spool_id = spool_response.json()["id"]
    
    # 3. Retrieve and verify extra fields preserved
    get_response = await client.get(f"/spool/{spool_id}")
    spool = get_response.json()
    
    import json
    stored_filament = json.loads(spool["extra"]["bambu_slicer_filament"])
    assert stored_filament == "GFSL05"
    
    stored_tag = json.loads(spool["extra"]["tag"])
    assert stored_tag == "A1B2C3D4E5F6G7H8"
```

### Compatibility Matrix

| Feature | Requirement | Notes |
|---------|-------------|-------|
| Vendor CRUD | `POST`, `GET`, `GET /{id}` | Basic lookup |
| Filament CRUD | `POST`, `GET`, `GET /{id}`, `PATCH` | Create/update filament types |
| Spool CRUD | `POST`, `GET`, `GET /{id}`, `PATCH` | Full lifecycle |
| Extra Fields | Registration + MERGE | Critical for Bambuddy |
| JSON Encoding | All extra values | Exact Spoolman behavior |
| Tag Support | `extra.tag` field | For RFID linking |

---

## References

### Source Code Files (from maziggy/bambuddy analysis)

- `backend/app/models/spoolman_slot_assignment.py` - Local slot assignment model
- `backend/app/api/routes/spoolman.py` - Bambuddy's Spoolman integration
- `backend/app/api/routes/spoolman_inventory.py` - Inventory management
- `backend/app/api/routes/_spoolman_helpers.py` - Helper functions for mapping
- `backend/app/utils/filament_ids.py` - ID conversion utilities
- `backend/app/core/database.py` - Database migrations & setup

### External References

- [Spoolman GitHub](https://github.com/Donkie/Spoolman)
- [Bambuddy GitHub](https://github.com/maziggy/bambuddy)
- [Bambu Lab Printer Documentation](https://bambulab.com/)
- [MQTT Protocol (3.1.1)](https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/mqtt-v3.1.1.html)

### Related Issues & PRs

- Bambuddy Issue #1326: Store default slicer profile on filament
- Bambuddy Issue #1329: Separate Slicer API and local slicer options
- Bambuddy PR #1114: Slicer filament round-trip fix

---

## Appendix: Conversion Function Reference

```python
# backend/app/utils/filament_ids.py (reference)

GENERIC_FILAMENT_IDS = {
    "PLA": "GFL99",
    "PETG": "GFG99",
    "ABS": "GFB99",
    "PC": "GFC99",
    "PA": "GFN99",
    "TPU": "GFU99",
    "PLA-CF": "GFL98",
    "PETG-CF": "GFG98",
    "PETG HF": "GFG96",
}

def setting_id_to_filament_id(setting_id: str) -> str:
    """GFSL05 → GFL05, PFUS... → PFUS..."""
    if setting_id.startswith("GFS"):
        return f"GF{setting_id[3:]}"
    return setting_id

def filament_id_to_setting_id(filament_id: str) -> str:
    """GFL05 → GFSL05, PFUS... → PFUS..."""
    if filament_id.startswith("GF") and filament_id[2] != "S":
        return f"GFS{filament_id[2:]}"
    return filament_id

def normalize_slicer_filament(value: str) -> tuple[str, str]:
    """Returns (tray_info_idx, setting_id) with version suffix stripped."""
    base = value.split("_")[0] if "_" in value else value
    return (setting_id_to_filament_id(base), filament_id_to_setting_id(base))
```

---

**End of Document**

*For questions or clarifications about this compatibility guide, refer to the source Bambuddy codebase or open an issue on the OpenSpoolMan project.*
