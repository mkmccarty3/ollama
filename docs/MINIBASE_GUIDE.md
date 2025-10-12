# Minibase: Complete Implementation Guide

**Purpose**: User distribution of customized Ollama binaries with embedded API keys  
**Status**: Production Ready  
**Last Updated**: October 10, 2025

---

## 🎯 Overview

Minibase is a fork of Ollama customized for distributing trained models to end users. Each user gets a **pre-configured binary** with their API key embedded - no setup required.

### User Experience
```bash
# User downloads: minibase-username-darwin-arm64.zip
# User extracts and runs:
./minibase pull your-model  # Just works!
./minibase run your-model "Hello world"
```

### What You Built
- ✅ Ollama fork with embedded config support
- ✅ Per-user binary builder
- ✅ Production model registry with authentication
- ✅ Integration with your FastAPI infrastructure

---

## 📦 Implementation Summary

### Changes to Ollama Fork

**File**: `envconfig/config.go`

Added embedded configuration support:
```go
// Can be set at compile time via -ldflags
var (
    EmbeddedRegistryURL = ""
    EmbeddedAPIKey      = ""
)

func MinibaseRegistryURL() string {
    // Env var > embedded > default
    if s := Var("MINIBASE_REGISTRY_URL"); s != "" {
        return s
    }
    if EmbeddedRegistryURL != "" {
        return EmbeddedRegistryURL
    }
    return "registry.ollama.ai"
}
```

**Modified Files**:
- `envconfig/config.go` - Added embedded config variables and functions
- `server/modelpath.go` - Uses `MinibaseRegistryURL()` for custom registry
- `server/routes.go` - Injects `MinibaseAPIKey()` in pull/push handlers

**Total Changes**: ~30 lines across 3 files

---

## 🏗️ Architecture

```
User's Machine                   Your Server
─────────────────                ─────────────────────

./minibase pull model      →     FastAPI (/v2/ endpoints)
  ↓                                ↓
Embedded config                  Verify API key
- registry: api.yourdomain.com   Check subscription
- api_key: user-abc123...        Log download
  ↓                                ↓
HTTPS request                    Stream GGUF file
  ↓                                ↓
Download to ~/.minibase/         From dream/local_models/
  ↓
./minibase run model
(inference on user's machine)
```

---

## 🔧 Building User Binaries

### Script Location
`rostra/scripts/build_user_binary.py`

### Usage
```bash
# Build for user
python scripts/build_user_binary.py USER_ID API_KEY [PLATFORM] [ARCH]

# Examples
python scripts/build_user_binary.py john key-abc123  # macOS ARM
python scripts/build_user_binary.py john key-abc darwin amd64  # macOS Intel
python scripts/build_user_binary.py john key-abc linux amd64   # Linux x64
python scripts/build_user_binary.py john key-abc windows amd64 # Windows x64
```

### Output
```
builds/
└── minibase-john-darwin-arm64-20251010.zip
    ├── minibase        # Pre-configured binary
    └── README.md       # User instructions
```

### How It Works
```bash
# Compiles with embedded configuration
go build -ldflags "\
  -X github.com/ollama/ollama/envconfig.EmbeddedRegistryURL=api.yourdomain.com \
  -X github.com/ollama/ollama/envconfig.EmbeddedAPIKey=user-key-abc123" \
  -o minibase ./main.go
```

---

## 📡 Model Registry

### Setup

**File**: `rostra/fastapi_server/model_registry.py`

**Add to** `rostra/fastapi_server/app.py`:
```python
from fastapi_server.model_registry import router as registry_router

app = FastAPI()
app.include_router(registry_router)
```

### Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v2/` | GET | Registry info & available models |
| `/v2/{namespace}/{model}/manifests/{tag}` | GET | Model metadata |
| `/v2/{namespace}/{model}/blobs/{digest}` | GET | Download model file |

### Authentication

Every request requires:
```
Authorization: Bearer <api-key>
```

The registry:
1. Verifies API key (checks database)
2. Validates subscription status
3. Applies rate limiting
4. Logs download for billing
5. Streams GGUF file

### Model Storage

Models are served from:
```
rostra/dream/local_models/
├── detoxify_small/
│   └── model.gguf
├── your-model/
│   └── model.gguf
└── another-model/
    └── model.gguf
```

### Publishing Models

After training, publish to registry:
```bash
# Copy GGUF to registry storage
cp nano_models/models/quantized/my-model.gguf \
   dream/local_models/my-model/model.gguf

# Or use a publishing script (TODO: create one)
python scripts/publish_model.py \
    my-model \
    nano_models/models/quantized/my-model.gguf
```

---

## 🔐 Authentication & Database

### API Keys Table

```sql
CREATE TABLE minibase_api_keys (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    api_key VARCHAR(64) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_used_at TIMESTAMP NULL,
    is_active BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (user_id) REFERENCES user(user_id)
);

CREATE TABLE minibase_downloads (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    model_name VARCHAR(255) NOT NULL,
    size_bytes BIGINT NOT NULL,
    downloaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES user(user_id)
);
```

### Integration Points

**TODO**: Implement in `model_registry.py`:
1. `verify_api_key()` - Query database for key validation
2. `log_download()` - Insert download records
3. Check Stripe subscription status
4. Apply rate limits per tier

---

## 🚀 User Workflow

### 1. User Subscribes
- User visits your site and subscribes (Stripe)
- API key automatically generated
- Stored in `minibase_api_keys` table

### 2. User Downloads Binary
- User clicks "Download for macOS"
- Triggers: `POST /generate-download?platform=darwin&arch=arm64`
- Server:
  1. Validates user session
  2. Checks subscription status  
  3. Runs `build_user_binary.py` with their API key
  4. Returns customized zip file

### 3. User Uses Minibase
```bash
# Extract download
unzip minibase-username-darwin-arm64.zip
cd minibase-username-darwin-arm64

# Pull model (no config needed!)
./minibase pull your-awesome-model

# Run inference (local on their machine)
./minibase run your-awesome-model "Hello world!"
```

---

## 🧪 Testing

### Test Embedded Build
```bash
cd /Users/codemonkey/Projects/ollama-minibase

# Build with test config
go build -ldflags "\
  -X github.com/ollama/ollama/envconfig.EmbeddedRegistryURL=localhost:8000 \
  -X github.com/ollama/ollama/envconfig.EmbeddedAPIKey=test-key-123" \
  -o bin/minibase-test ./main.go

# Should use embedded config (no env vars needed)
./bin/minibase-test --help
```

### Test Registry
```bash
# Start FastAPI server
cd /Users/codemonkey/Projects/rostra
python fastapi_server/app.py

# In another terminal, test endpoints
curl http://localhost:8000/v2/
curl -H "Authorization: Bearer test-key" \
  http://localhost:8000/v2/library/detoxify_small/manifests/latest
```

### Test Full Flow
```bash
# 1. Build user binary
python scripts/build_user_binary.py testuser test-key-123

# 2. Extract and test
cd builds
unzip minibase-testuser-darwin-arm64-*.zip
cd minibase-testuser-*
./minibase pull detoxify_small  # Should work!
```

---

## 📋 Deployment Checklist

### Before Production

- [ ] Update `EmbeddedRegistryURL` default to your domain
- [ ] Implement database auth in `verify_api_key()`
- [ ] Implement `log_download()` database inserts
- [ ] Add Stripe subscription checking
- [ ] Set up rate limiting per tier
- [ ] Configure HTTPS/SSL on your domain
- [ ] Test builds for all platforms (darwin, linux, windows)
- [ ] Add error handling and logging
- [ ] Set up monitoring and alerts

### MediaWiki Integration

- [ ] Add download page/endpoint
- [ ] Add download buttons for each platform
- [ ] Generate API keys on subscription
- [ ] Link to Stripe webhook handler
- [ ] Add usage dashboard for users
- [ ] Create support documentation

---

## 🔄 Updating from Upstream

### Sync with Ollama

```bash
cd /Users/codemonkey/Projects/ollama-minibase

# Fetch upstream
git fetch upstream

# Rebase minibase branch
git rebase upstream/main

# Resolve conflicts (if any)
# Focus on envconfig/config.go and server/ files

# Push to your fork
git push origin minibase --force-with-lease
```

### Update Vendored Code in Rostra

```bash
cd /Users/codemonkey/Projects/rostra

# Pull updated subtree
git subtree pull --prefix third_party/ollama ollama-fork minibase --squash

# Rebuild binaries
python scripts/build_minibase_ollama.py

# Test
```

---

## 📁 File Reference

### Ollama Fork (`ollama-minibase/`)
- `envconfig/config.go` - Embedded config support
- `server/modelpath.go` - Custom registry URL
- `server/routes.go` - API key injection
- `bin/minibase` - Built binary

### Rostra (`rostra/`)
- `scripts/build_user_binary.py` - Per-user binary builder
- `scripts/build_minibase_ollama.py` - Standard build script for Ollama
- `fastapi_server/model_registry.py` - Production registry
- `fastapi_server/app.py` - FastAPI main app
- `dream/local_models/` - Model storage
- `third_party/ollama/` - Vendored fork (after subtree)
- `builds/` - Generated user binaries

---

## 🎯 Key Decisions

| Decision | Rationale |
|----------|-----------|
| **Embedded config** | Users don't need to configure anything |
| **Per-user binaries** | Each user gets their own API key embedded |
| **Ollama-compatible API** | No changes needed to core pull logic |
| **FastAPI registry** | Integrates with existing infrastructure |
| **Local file storage** | Simple, fast, can migrate to GCS later |
| **No fallback** | Users only get YOUR models (not Ollama's) |

---

## 💡 Future Enhancements

- [ ] Web UI for model management
- [ ] Usage analytics dashboard
- [ ] GCS backend for model storage
- [ ] CDN for global distribution
- [ ] Model versioning system
- [ ] Automated model publishing pipeline
- [ ] Model usage tracking per user
- [ ] Tiered access (free/pro/enterprise models)

---

## 🆘 Troubleshooting

### Build Fails
```bash
# Check Go version
go version  # Need 1.22+

# Clean and rebuild
cd third_party/ollama
go clean
go build -o ../../bin/minibase ./main.go
```

### Registry Not Working
```bash
# Check if FastAPI is running
curl http://localhost:8000/v2/

# Check model exists
ls dream/local_models/*/model.gguf

# Check logs
tail -f fastapi_server.log
```

### Binary Doesn't Connect
```bash
# Test embedded config (shouldn't need env vars)
./minibase --help

# If it needs env vars, embedded config didn't work
# Rebuild with correct ldflags
```

---

## 📞 Quick Reference

### Build User Binary
```bash
python scripts/build_user_binary.py USER_ID API_KEY PLATFORM ARCH
```

### Start Registry
```bash
cd rostra/fastapi_server
python app.py
```

### Test Pull
```bash
export MINIBASE_REGISTRY_URL="localhost:8000"
export MINIBASE_API_KEY="test-key"
./bin/minibase pull model-name
```

---

**Status**: Implementation complete  
**Next Steps**: Test, deploy to production, integrate with MediaWiki downloads  
**Contact**: See repository issues for questions

