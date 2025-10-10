# Minibase Ollama Integration - Implementation Plan

**Date**: October 10, 2025  
**Status**: AWAITING APPROVAL

---

## 🔍 Research Summary

### What I Found in Your Code:

**Database Schema**:
- `training_models` table: Stores all models with `org_id`, `user_id`, `model_id`, `status`, `privacy`, `gcs_artifact_uri`
- `api_keys` table: Stores API keys with `org_id`, `user_id`, `key_prefix`, `key_hash`
- `organization_users` table: Links users to organizations

**Model Ownership Logic** (from ModelPackaging/ApiDownloadPackage.php):
```php
// User owns model if:
org_id = user_org AND user_id = user_id AND deleted = 0

// OR user can access team/public models:
org_id = user_org AND privacy IN ('team', 'public') AND deleted = 0

// Private models: only owner can access
```

**Model Storage**:
- Models stored in Google Cloud Storage at `gcs_artifact_uri`
- Example: `gs://bucket-name/path/to/model.gguf`
- Downloaded via Google\Cloud\Storage\StorageClient PHP library
- NEVER expose GCS URIs directly to users

**API Key Validation** (from ApiKeyAuth extension):
- Keys extracted from `Authorization: Bearer <key>` header
- Verified via `ApiKeyStore::verifyKey($keyPrefix, $fullKey)`
- Returns `['org_id' => int, 'user_id' => int]` on success
- Checks for revoked/expired keys

**Current Download Flow** (ModelPackaging):
1. User requests download via PHP API
2. `validateModelAccess()` checks ownership/permissions based on org_id + user_id
3. Gets `gcs_artifact_uri` from `training_models` table
4. Downloads GGUF from GCS using PHP StorageClient
5. Packages and streams to user via PHP headers

---

## ❌ What I Built Wrong

1. **FastAPI Registry** - Should be PHP API within MediaWiki extensions
2. **dream/local_models/** - Models are in GCS, not local filesystem
3. **No Ownership Checking** - Just checked if API key exists, didn't check model ownership
4. **New Tables** - Should use existing `training_models` and `api_keys` tables
5. **Wrong Authentication** - Should use existing `ApiKeyStore::verifyKey()`

---

## ✅ Correct Implementation Plan

### Phase 1: Create OllamaRegistry MediaWiki Extension

**Location**: `rostra/mediawiki/extensions/OllamaRegistry/`

**Purpose**: Provide Ollama-compatible registry endpoints that:
- List models user owns (based on `org_id` + `user_id` from API key)
- Return model manifests (metadata)
- Stream model blobs (GGUF files from GCS)

**Files to Create**:

```
OllamaRegistry/
├── extension.json
├── includes/
│   ├── ApiListModels.php       # List user's models
│   ├── ApiGetManifest.php      # Get model manifest
│   ├── ApiGetBlob.php          # Download model GGUF
│   └── OllamaAuthHelper.php    # API key validation
├── i18n/
│   └── en.json
└── README.md
```

### Phase 2: Implement API Endpoints

#### Endpoint 1: List Models (`action=ollama_listModels`)

**Purpose**: Return list of models user can access

**Request**:
```
POST /api.php
action=ollama_listModels
Authorization: Bearer <api-key>
```

**Logic**:
1. Extract API key from `Authorization` header
2. Call `ApiKeyStore::verifyKey()` to get `org_id` and `user_id`
3. Query `training_models` table:
   ```sql
   SELECT model_id, model_name, description, artifact_size_mb, status, privacy, updated_time
   FROM training_models
   WHERE org_id = ? 
     AND deleted = 0
     AND status IN ('ready', 'trained', 'deployed')
     AND (
       user_id = ?  -- Models user owns
       OR privacy IN ('team', 'public')  -- Team/public models
     )
   ORDER BY updated_time DESC
   ```
4. Return JSON array of models

**Response**:
```json
{
  "models": [
    {
      "model_id": "my-model-123",
      "model_name": "My Awesome Model",
      "description": "...",
      "size_mb": 145,
      "privacy": "private",
      "updated_time": "2025-10-10 12:00:00"
    }
  ]
}
```

#### Endpoint 2: Get Manifest (`action=ollama_getManifest`)

**Purpose**: Return Ollama-compatible manifest for a specific model

**Request**:
```
POST /api.php
action=ollama_getManifest
model_id=my-model-123
Authorization: Bearer <api-key>
```

**Logic**:
1. Extract API key, get `org_id` and `user_id`
2. Query `training_models` with ownership check (same logic as ModelPackaging)
3. Get `gcs_artifact_uri` and `artifact_size_mb`
4. Calculate SHA256 digest (or use cached digest if available)
5. Return Ollama-compatible manifest

**Response**:
```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "config": {
    "mediaType": "application/vnd.ollama.model.config.v1+json",
    "digest": "sha256:abc123...",
    "size": 152428800
  },
  "layers": [
    {
      "mediaType": "application/vnd.ollama.model.layer.v1",
      "digest": "sha256:abc123...",
      "size": 152428800
    }
  ]
}
```

#### Endpoint 3: Get Blob (`action=ollama_getBlob`)

**Purpose**: Stream GGUF file from GCS

**Request**:
```
POST /api.php
action=ollama_getBlob
model_id=my-model-123
digest=sha256:abc123...
Authorization: Bearer <api-key>
```

**Logic**:
1. Extract API key, get `org_id` and `user_id`
2. Query `training_models` with ownership check
3. Get `gcs_artifact_uri` 
4. Download from GCS (like ModelPackaging does)
5. Stream to user with proper headers

**Response**:
```
Content-Type: application/octet-stream
Content-Length: 152428800
Content-Disposition: attachment; filename="model.gguf"

<binary GGUF data streamed>
```

### Phase 3: Update Ollama Fork to Use Custom Endpoints

**Current Code** (envconfig/config.go):
- Already has `EmbeddedRegistryURL` and `EmbeddedAPIKey`
- Already uses these in `server/modelpath.go` and `server/routes.go`

**What Needs to Change**:
- Map Ollama's `/v2/{namespace}/{model}/manifests/{tag}` requests
- To MediaWiki's `action=ollama_getManifest&model_id={model}`

**How**: Ollama expects URLs like:
```
https://registry.ollama.ai/v2/library/llama2/manifests/latest
```

Your server will need to handle:
```
https://yourdomain.com/api.php?action=ollama_getManifest&model_id=llama2
```

**Options**:
1. **Option A** (Recommended): Use Apache/Nginx rewrite rules to map paths
   ```apache
   # In .htaccess or Apache config
   RewriteRule ^v2/([^/]+)/([^/]+)/manifests/(.+)$ /api.php?action=ollama_getManifest&namespace=$1&model=$2&tag=$3 [QSA,L]
   RewriteRule ^v2/([^/]+)/([^/]+)/blobs/(.+)$ /api.php?action=ollama_getBlob&namespace=$1&model=$2&digest=$3 [QSA,L]
   ```

2. **Option B**: Modify Ollama fork to call MediaWiki API format directly
   - More invasive changes to Ollama code
   - Less standard

**Recommendation**: Option A with URL rewriting

### Phase 4: Update Binary Builder

**File**: `rostra/scripts/build_user_binary.py`

**Current**:
- Embeds `api.yourdomain.com` as registry URL
- Embeds user's API key

**What Needs to Change**:
- Registry URL should be: `https://yourdomain.com` (your MediaWiki domain)
- API key comes from `api_keys` table (user's real API key)

**New Flow**:
1. User subscribes/signs up
2. API key auto-generated via ApiKeyAuth extension (already exists)
3. User clicks "Download Minibase"
4. Server runs `build_user_binary.py` with:
   - user_id → username
   - api_key → from `api_keys` table
   - registry_url → `https://yourdomain.com`
5. Binary is built with embedded config
6. User downloads and runs

### Phase 5: Replace ModelPackaging Downloads

**Current**: ModelPackaging extension provides:
- macOS app bundles
- Raw GGUF downloads
- Download UI modal

**New Approach**:
- Keep ModelPackaging for legacy downloads OR
- Replace download button with "Download Minibase" button
- This button generates customized Ollama binary

**UI Flow**:
```
User on Special:ApiKeys page:
  ├─ Sees button at top: "Download Minibase Ollama"
  ├─ Clicks button → Modal opens with platform selection
  ├─ Selects platform (e.g., macOS Apple Silicon)
  ├─ Clicks "Download" → Button shows loading state
  └─ Waits 30-60 seconds → Download starts automatically

After download:
  User extracts and runs: ./minibase list
  ↓
  Shows all their models
  ↓
  User runs: ./minibase pull my-model-123
  ↓
  Minibase connects to yourdomain.com/api.php
  ↓
  Uses embedded API key for auth
  ↓
  Downloads GGUF file
```

**Decision**: ✅ **Option A - Keep ModelPackaging as-is, add Ollama download on API Keys page**

This means:
- ModelPackaging remains unchanged (existing GGUF/macOS app downloads on model pages)
- NEW: "Download Minibase Ollama" button on Special:ApiKeys page
- User downloads binary once (tied to their API key)
- Binary can pull ANY of their models (no need to download per-model)
- Clear separation: API Keys page = get CLI tool, Model pages = direct downloads

### Phase 6: Handle Model Import from Marketplace

**From Your Code**: Users can import models from marketplace

**Implication**: When user imports a model:
1. New row created in `training_models` with `is_imported=1`
2. References original via `original_owner_id` and `original_org_id`
3. Privacy determines visibility

**Ollama Integration**: When user runs `./minibase list`:
- Shows ALL models they own (including imported ones)
- Each shows up as `minibase://their-org/model-name`
- User can pull any of them

---

## 📋 Files to Delete/Revert

### Delete These (wrong implementation):
```
rostra/fastapi_server/model_registry.py
rostra/scripts/build_user_binary.py (will recreate correctly)
```

### Keep/Update These:
```
ollama-minibase/envconfig/config.go (embedded config is correct)
ollama-minibase/server/modelpath.go (already correct)
ollama-minibase/server/routes.go (already correct)
```

---

## 🔄 Step-by-Step Implementation Order

### Phase 1: Database Migration
1. **Add artifact_digest column** to training_models table
2. **Create backfill script** (`scripts/backfill_artifact_digests.php`)
3. **Run backfill** for existing models (may take time for large models)
4. **Update ApiProcessModel.php** to calculate digest during quantization

### Phase 2: Create OllamaRegistry Extension
1. **Extension structure**:
   - extension.json
   - ApiListModels.php
   - ApiGetManifest.php
   - ApiGetBlob.php
   - OllamaAuthHelper.php (wraps ApiKeyStore)
2. **Enable extension** in LocalSettings.php

### Phase 3: URL Rewrite Rules
1. **Add to .htaccess** or Apache VirtualHost config:
   ```apache
   # Ollama registry compatibility
   RewriteRule ^v2/?$ /api.php?action=ollama_info [QSA,L]
   RewriteRule ^v2/([^/]+)/([^/]+)/manifests/(.+)$ /api.php?action=ollama_getManifest&namespace=$1&model=$2&tag=$3 [QSA,L]
   RewriteRule ^v2/([^/]+)/([^/]+)/blobs/(.+)$ /api.php?action=ollama_getBlob&namespace=$1&model=$2&digest=$3 [QSA,L]
   ```
2. **Test rewrites** with curl

### Phase 4: Binary Builder API
1. **Create API endpoint**: `ApiGenerateOllamaBinary.php` in ApiKeyAuth extension
   - Module name: `apikey_generateOllamaBinary`
   - Parameters: platform, arch
   - Gets authenticated user from session
   - Retrieves user's API key from api_keys table (most recently created, non-revoked)
   - Validates ollama-minibase source path exists on server
   - Builds with embedded config:
     ```php
     $ldflags = "-X github.com/ollama/ollama/envconfig.EmbeddedRegistryURL={$registryUrl} " .
                "-X github.com/ollama/ollama/envconfig.EmbeddedAPIKey={$apiKey}";
     ```
   - Creates zip with binary + README
   - Streams to user with proper headers
2. **Server Requirements**:
   - Go 1.22+ installed on web server
   - ollama-minibase source cloned at known path (e.g., `/opt/ollama-minibase/`)
   - Write permissions to temp directory for builds
3. **Build script helper** (optional): Shell script to handle Go compilation

### Phase 5: Download UI (API Keys Page)
1. **Extend SpecialApiKeys.php** (ApiKeyAuth extension):
   - Add button at top of page: "Download Minibase Ollama"
   - Style consistent with existing buttons on the page
   - Opens modal/dropdown with platform selection:
     - macOS (Apple Silicon / Intel)
     - Linux (ARM64 / x64)
     - Windows (x64)
   - "Download" button triggers ApiGenerateOllamaBinary
   - Button enters loading state during build (30-60 seconds)
   - Shows message: "Generating your custom binary... This may take a minute."
2. **JavaScript** (`ext.ApiKeyAuth.apikeys.js`):
   - Handle download button click
   - Show platform selection UI
   - Trigger API call to generate binary
   - Display loading state with spinner
   - Handle download when ready
3. **CSS** (`ext.ApiKeyAuth.apikeys.css`):
   - Style download button and modal
   - Loading state animation

### Phase 6: Testing
1. **Unit tests**:
   - Test model ownership queries
   - Test API key validation
   - Test manifest generation
2. **Integration tests**:
   - Create test user with API key
   - Train/import test model
   - Generate binary
   - Run `./minibase list`
   - Run `./minibase pull model-name`
   - Verify GGUF downloads correctly
3. **Cross-platform tests**:
   - Test macOS binary
   - Test Linux binary
   - Test Windows binary

---

## ❓ Questions for You

### Critical Decisions Needed:

1. **Registry URL Format**: 
   - ✅ **ANSWERED: Use Apache RewriteRule** to map `/v2/` paths to MediaWiki API actions
   - This is better for upstream compatibility (fewer changes to Ollama fork)
   - Rewrite rules handle URL translation transparently

2. **ModelPackaging Integration**:
   - ✅ **ANSWERED: Keep ModelPackaging unchanged**
   - Add Ollama download button on Special:ApiKeys page (not on model pages)
   - One binary download per user (can pull all their models)

3. **Model Naming in Ollama**:
   - ✅ **ANSWERED: Use `model_id` directly** (e.g., `my-model-123`)
   - Simple format: `minibase pull my-model-123`
   - Can enhance with org/name format later if needed

4. **Digest Caching**:
   - ✅ **ANSWERED: Pre-calculate and cache in database**
   - Add `artifact_digest` column to `training_models` table
   - Calculate during quantization/processing step
   - **REQUIRED**: Create migration script to backfill existing models

5. **Binary Distribution**:
   - ✅ **ANSWERED: On-demand per-user builds**
   - Build when user requests download (30-60 second wait)
   - Each binary has unique API key embedded at compile time
   - Stream directly to user, no caching (can't share between users)
   - Show "Generating your custom binary..." spinner in UI

---

## 📦 Database Changes Needed

### Add Digest Caching (REQUIRED):

**Migration SQL**:
```sql
ALTER TABLE training_models 
ADD COLUMN artifact_digest VARCHAR(71) NULL COMMENT 'SHA256 digest (sha256:hash) of artifact for Ollama registry';

-- Index for lookups
CREATE INDEX idx_artifact_digest ON training_models(artifact_digest);
```

**Backfill Script**: `rostra/scripts/backfill_artifact_digests.php`
- Iterates through all models with `status IN ('ready', 'trained', 'deployed')`
- Downloads GGUF from GCS
- Calculates SHA256 digest
- Updates `artifact_digest` column
- Run once after migration

**Integration**: Update `ApiProcessModel.php` to calculate digest during quantization step

---

## 🎯 Expected Outcome

**User Experience**:
```bash
# User downloads minibase-john-macos-arm64.zip
# Extracts it
./minibase list
# Shows:
# - my-private-model
# - team-shared-model  
# - imported-marketplace-model

./minibase pull my-private-model
# Downloads from yourdomain.com
# Saves to ~/.minibase/models/

./minibase run my-private-model "Hello!"
# Runs inference locally
```

**No configuration needed!** API key and registry URL are embedded.

---

## ⏱️ Estimated Implementation Time

- Database migration + backfill script: 2-3 hours
- Create OllamaRegistry extension: 3-4 hours
- Add URL rewrites: 30 minutes
- Create binary builder API: 2-3 hours  
- Add download UI: 2-3 hours
- Testing: 2-3 hours
- **Total**: 12-16 hours

---

## 🚦 Implementation Summary - All Questions Answered

### ✅ Decisions Made:
1. **Registry URL**: Apache RewriteRule (better for upstream compatibility)
2. **ModelPackaging**: Keep existing + add Ollama as new option
3. **Model Naming**: Use `model_id` directly (simple, can enhance later)
4. **Digest Caching**: Database column with backfill script
5. **Binary Distribution**: On-demand per-user builds (30-60s wait)
6. **Source Location**: Keep separate, access during build

### 📋 Deliverables:
1. ✅ OllamaRegistry MediaWiki extension (PHP)
2. ✅ Apache rewrite rules for /v2/ paths
3. ✅ Database migration + backfill script
4. ✅ Binary builder API endpoint (in ApiKeyAuth extension)
5. ✅ Updated SpecialApiKeys page with download button
6. ✅ JavaScript for platform selection and loading state
7. ✅ Testing suite
8. ✅ Deployment documentation (including Go installation on server)

### 🎯 User Experience:
```bash
# User navigates to Special:ApiKeys
# Clicks "Download Minibase Ollama" button at top of page
# Selects "macOS (Apple Silicon)"
# Clicks "Download" → Button shows loading state
# Waits 30-60 seconds while binary builds
# Downloads: minibase-macos-arm64.zip

# Extracts and runs:
./minibase list
# Shows all their models (private, team, imported)

./minibase pull my-model-123
# Downloads from yourdomain.com with embedded API key
# Saves to ~/.minibase/models/

./minibase run my-model-123 "Hello!"
# Runs inference locally
```

---

## 🚀 Ready to Proceed?

**All questions answered. Approve to begin implementation.**

Once approved, I will:
1. Delete wrong files (fastapi_server/model_registry.py, scripts/build_user_binary.py)
2. Create database migration SQL
3. Create backfill script
4. Create OllamaRegistry extension with all endpoints
5. Create binary builder API in ApiKeyAuth extension
6. Update SpecialApiKeys page with download button
7. Add JavaScript for platform selection modal and loading states
8. Test end-to-end
9. Provide deployment instructions (including Go setup on server)

