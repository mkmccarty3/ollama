# Minibase Ollama: Complete Implementation Guide

**Purpose**: User distribution of customized Ollama CLI with config file-based authentication  
**Status**: Production Ready  
**Last Updated**: October 13, 2025

---

## 🎯 Overview

Minibase Ollama is a fork of Ollama customized for distributing trained models to end users. Users download a package containing:
1. **Pre-built Ollama binary** (built via GitHub Actions for all platforms)
2. **config.json** with their personal API key and registry URL
3. **install script** for automatic setup

### User Experience
```bash
# User downloads: minibase-ollama-username-darwin-arm64.zip
# User runs install script:
./install.sh

# Everything is automatically configured!
ollama pull your-model  # Just works!
ollama run your-model "Hello world"
```

### What You Built
- ✅ Ollama fork with config file support (~/.minibase/config.json)
- ✅ GitHub Actions multi-platform binary builds (macOS/Linux/Windows)
- ✅ PHP download API for packaging binaries with user configs
- ✅ Production model registry with authentication
- ✅ Integration with MediaWiki/PHP infrastructure

---

## 📦 Implementation Summary

### Phase 1: Ollama Code Modifications

**File**: `envconfig/config.go`

Added config file support with priority hierarchy:
```go
// Config file structure
type MinibaseConfig struct {
    RegistryURL string `json:"registry_url"`
    APIKey      string `json:"api_key"`
    UserID      int    `json:"user_id,omitempty"`
    Username    string `json:"username,omitempty"`
    OrgID       int    `json:"org_id,omitempty"`
}

// Load from ~/.minibase/config.json
func loadMinibaseConfig() *MinibaseConfig {
    configPath := filepath.Join(home, ".minibase", "config.json")
    // ... load and parse JSON
    return &config
}

// Priority: env var > config file > embedded > default
func MinibaseRegistryURL() string {
    if s := Var("MINIBASE_REGISTRY_URL"); s != "" {
        return s  // 1. Environment variable (testing)
    }
    if config := loadMinibaseConfig(); config != nil {
        return config.RegistryURL  // 2. Config file (production)
    }
    if EmbeddedRegistryURL != "" {
        return EmbeddedRegistryURL  // 3. Embedded (deprecated)
    }
    return "registry.ollama.ai"  // 4. Default
}
```

**Modified Files**:
- `envconfig/config.go` - Config file loading, priority hierarchy
- `server/modelpath.go` - Uses `MinibaseRegistryURL()` for custom registry
- `server/routes.go` - Injects `MinibaseAPIKey()` in pull/push handlers

**Total Changes**: ~85 lines across 3 files

### Phase 2: GitHub Actions Workflow

**File**: `.github/workflows/build-minibase-ollama.yml`

Automated multi-platform builds:
- **macOS**: Intel (amd64) + Apple Silicon (arm64) on `macos-13-xlarge`
- **Linux**: x64 (amd64) + ARM64 (arm64) with cross-compilation
- **Windows**: x64 (amd64) with TDM-GCC

Workflow:
1. Build binaries for all platforms
2. Generate SHA256 checksums
3. Upload to GCS buckets:
   - `gs://minibase-ollama-binaries/latest/` (production)
   - `gs://minibase-ollama-binaries-staging/latest/` (staging)
4. Create versioned backups in `versions/`

Triggers:
- Automatic: Push to `minibase` branch → uploads to staging
- Manual: Workflow dispatch → choose staging/production/both

### Phase 3: PHP Download API

**File**: `rostra/mediawiki/extensions/OllamaRegistry/includes/ApiGenerateOllamaBinary.php`

Completely rewritten to:
1. Download pre-built binary from GCS
2. Generate config.json with user's API key
3. Create install script (Unix shell or Windows batch)
4. Generate comprehensive README
5. Package everything into a ZIP
6. Stream to user

**Key Features**:
- ⚡ Fast downloads (5-10 seconds vs 30-60 seconds)
- 🔐 Secure: New API key generated per download
- 🎯 User-specific: config.json contains registry URL + API key
- 📦 Complete package: Binary + config + install script + README
- 🔒 Concurrency protection: Lock file prevents duplicate requests

### Phase 4: UI Updates

**File**: `rostra/mediawiki/extensions/ApiKeyAuth/includes/SpecialApiKeys.php`

Added "Download Minibase Ollama" button with platform selector:
- macOS (Intel + Apple Silicon)
- Linux (x64 + ARM64)
- Windows (x64)

---

## 🏗️ Architecture

```
User's Machine                   Your Infrastructure
──────────────────               ─────────────────────────────

Download from:                   GitHub Actions:
api.minibase.ai                  - Builds binaries for all platforms
  ↓                              - Uploads to GCS buckets
                                 
Install:                         GCS Storage:
./install.sh                     - minibase-ollama-binaries/latest/
  ↓                              - Pre-built binaries ready
Copies to:                       
~/.minibase/                     PHP API:
  ├── bin/ollama                 - Downloads binary from GCS
  └── config.json                - Generates user config.json
                                 - Creates install scripts
User runs:                       - Packages and streams ZIP
ollama pull model
  ↓                              MediaWiki OllamaRegistry:
Reads config:                    - Validates API key
~/.minibase/config.json          - Checks model ownership
  ↓                              - Streams GGUF from GCS
Requests from:
api.minibase.ai/v2/
  ↓
Downloads model to:
~/.minibase/models/
  ↓
ollama run model
(inference on user's machine)
```

---

## 🔑 Configuration

### User Config File

Location: `~/.minibase/config.json`

```json
{
  "registry_url": "https://minibase.ai",
  "api_key": "your-api-key-here",
  "user_id": 123,
  "username": "yourname"
}
```

**Priority Hierarchy**:
1. `MINIBASE_REGISTRY_URL` env var (testing/override)
2. `~/.minibase/config.json` (production)
3. Compile-time embedded config (deprecated)
4. Default: `registry.ollama.ai`

### Custom Config Location

Set `MINIBASE_CONFIG_DIR` environment variable:
```bash
export MINIBASE_CONFIG_DIR=/path/to/config
# Ollama will look for /path/to/config/config.json
```

---

## 🚀 Deployment Flow

### 1. GitHub Actions Build (Automatic)

```bash
# On push to minibase branch:
git push origin minibase

# GitHub Actions automatically:
# 1. Builds binaries for all platforms
# 2. Uploads to gs://minibase-ollama-binaries-staging/latest/
# 3. Takes ~15-20 minutes
```

### 2. Manual Production Deploy

```bash
# Go to: https://github.com/mkmccarty3/ollama/actions
# Select "Build Minibase Ollama Binaries"
# Click "Run workflow"
# Choose: "production" or "both"
# Click "Run workflow"
```

### 3. User Downloads

Users visit: `https://minibase.ai/wiki/Special:ApiKeys`

Click "Download Minibase Ollama" → Select platform → Download starts

Package contains:
```
minibase-ollama-username-darwin-arm64.zip
├── ollama                  # Pre-built binary
├── config.json             # User's API key & registry URL
├── install.sh              # Automatic installer
└── README.md               # Installation & usage guide
```

---

## 📝 Installation Instructions (For Users)

### macOS / Linux

```bash
# 1. Extract the ZIP
unzip minibase-ollama-username-darwin-arm64.zip
cd minibase-ollama-username-darwin-arm64

# 2. Run install script
chmod +x install.sh
./install.sh

# 3. Reload shell
source ~/.bashrc  # or ~/.zshrc

# 4. Start using!
ollama pull model-name
ollama run model-name
```

### Windows

```batch
REM 1. Extract the ZIP
REM 2. Double-click install.bat
REM 3. Follow prompts
REM 4. Use:
ollama pull model-name
ollama run model-name
```

### Manual Installation

```bash
# 1. Create config directory
mkdir -p ~/.minibase

# 2. Copy config
cp config.json ~/.minibase/config.json

# 3. Copy binary
mkdir -p ~/.minibase/bin
cp ollama ~/.minibase/bin/ollama
chmod +x ~/.minibase/bin/ollama

# 4. Add to PATH (add to ~/.bashrc or ~/.zshrc)
export PATH="$HOME/.minibase/bin:$PATH"
```

---

## 🗄️ GCS Bucket Structure

### Production: `minibase-ollama-binaries`

```
minibase-ollama-binaries/
├── latest/
│   ├── ollama-darwin-amd64
│   ├── ollama-darwin-arm64
│   ├── ollama-linux-amd64
│   ├── ollama-linux-arm64
│   ├── ollama-windows-amd64.exe
│   ├── SHA256SUMS
│   └── manifest.json
└── versions/
    ├── v0.1.0-minibase_20251013_143022/
    └── ... (historical backups)
```

### Staging: `minibase-ollama-binaries-staging`

Same structure, used for testing before production release.

---

## 🔒 Security & Permissions

### GCS Service Accounts

**GitHub Actions Upload SA**: `github-actions-ollama@wikihealthy.iam.gserviceaccount.com`
- Permission: `storage.objectCreator` on both buckets
- Purpose: Upload binaries from GitHub Actions

**Server Download SA**: `658802496686-compute@developer.gserviceaccount.com`
- Permission: `storage.objectViewer` on both buckets
- Purpose: Download binaries for user packages

### API Keys

Each download generates a **new API key**:
- Stored in database: `api_keys` table (only hash stored)
- Full key: Given to user in `config.json`
- Name: `"Ollama CLI - YYYY-MM-DD HH:MM:SS"`
- Unique per download (not reused)

---

## 🧪 Testing

### Test Config File Loading

```bash
# Create test config
mkdir -p ~/.minibase
cat > ~/.minibase/config.json << EOF
{
  "registry_url": "https://staging.minibase.ai",
  "api_key": "test-key-12345"
}
EOF

# Test Ollama recognizes it
ollama list
# Should attempt to connect to staging.minibase.ai
```

### Test Different Platforms

Download and test on:
- ✅ macOS Intel (darwin/amd64)
- ✅ macOS Apple Silicon (darwin/arm64)
- ✅ Linux x64 (linux/amd64)
- ✅ Linux ARM64 (linux/arm64)
- ✅ Windows x64 (windows/amd64)

### Verify Install Script

```bash
# Run install script
./install.sh

# Verify files
ls -la ~/.minibase/
# Should see: bin/ and config.json

# Verify PATH
which ollama
# Should be: ~/.minibase/bin/ollama

# Test command
ollama --version
```

---

## 🐛 Troubleshooting

### Binary Not Found After Install

**Problem**: `ollama: command not found`

**Solution**:
```bash
# Reload shell
source ~/.bashrc  # or ~/.zshrc

# Or use full path
~/.minibase/bin/ollama list
```

### Permission Denied (macOS/Linux)

**Problem**: `Permission denied` when running ollama

**Solution**:
```bash
chmod +x ~/.minibase/bin/ollama
```

### API Key Errors

**Problem**: `Unauthorized` or `403` errors

**Solution**:
```bash
# Check config
cat ~/.minibase/config.json

# Regenerate key
# Visit: https://minibase.ai/wiki/Special:ApiKeys
# Click "Download Minibase Ollama" again
```

### Models Not Downloading

**Problem**: `Failed to pull model`

**Causes**:
1. Model doesn't exist
2. User doesn't own model
3. API key expired
4. Network issues

**Solution**:
```bash
# Check which models you own
ollama list

# Test connection
curl -H "Authorization: Bearer YOUR_API_KEY" \
  https://minibase.ai/v2/

# Check logs
tail -f ~/.minibase/logs/ollama.log
```

---

## 🔄 Maintenance

### Update Ollama Version

```bash
# 1. Pull latest from upstream Ollama
cd ollama-minibase
git fetch upstream
git merge upstream/main

# 2. Resolve conflicts (if any)
# 3. Push to trigger build
git push origin minibase

# 4. Wait for GitHub Actions build
# 5. Test staging downloads
# 6. Deploy to production
```

### Clean Up Old Versions

```bash
# Keep only last 10 versions
gsutil ls gs://minibase-ollama-binaries/versions/ | \
  sort -r | tail -n +11 | \
  xargs -I {} gsutil -m rm -r {}
```

### Monitor Downloads

```bash
# Check PHP logs
tail -f /var/log/mediawiki/ollamabinary.log

# Check GCS access logs
gsutil logging get gs://minibase-ollama-binaries
```

---

## 📊 Metrics

### Binary Sizes

- **macOS ARM64**: ~45 MB
- **macOS Intel**: ~47 MB
- **Linux x64**: ~43 MB
- **Linux ARM64**: ~41 MB
- **Windows x64**: ~44 MB

### Build Times (GitHub Actions)

- **macOS builds**: ~8-10 minutes each
- **Linux builds**: ~5-7 minutes each
- **Windows build**: ~10-12 minutes
- **Total workflow**: ~15-20 minutes

### Download Times

- **Binary download from GCS**: ~1-2 seconds
- **Config generation**: <1 second
- **ZIP packaging**: <1 second
- **Total user experience**: 5-10 seconds

---

## 🎓 Advanced Usage

### Override Registry URL (Testing)

```bash
# Temporarily use staging
export MINIBASE_REGISTRY_URL=https://staging.minibase.ai
ollama pull test-model

# Permanently (add to ~/.bashrc)
echo 'export MINIBASE_REGISTRY_URL=https://staging.minibase.ai' >> ~/.bashrc
```

### Multiple Config Locations

```bash
# Work config
export MINIBASE_CONFIG_DIR=~/.minibase-work
ollama pull work-model

# Personal config
export MINIBASE_CONFIG_DIR=~/.minibase-personal
ollama pull personal-model
```

### Update API Key Without Reinstalling

```bash
# Edit config
vi ~/.minibase/config.json
# Change "api_key" value

# Test
ollama list
```

---

## 📚 Related Documentation

- **Apache Rewrite Rules**: `rostra/mediawiki/extensions/OllamaRegistry/APACHE_REWRITE_RULES.md`
- **Staging/Production Setup**: `ollama-minibase/STAGING_PRODUCTION_SETUP.md`
- **Ollama API Details**: `ollama-minibase/OLLAMA_API_DETAILS.md`

---

## ✅ Success Criteria

You know it's working when:
- ✅ GitHub Actions builds succeed for all platforms
- ✅ Binaries appear in GCS `latest/` directory
- ✅ Download button works on API Keys page
- ✅ ZIP contains binary + config + install script + README
- ✅ Install script creates `~/.minibase/` structure
- ✅ `ollama pull model-name` downloads your models
- ✅ `ollama run model-name` executes inference locally

---

**Congratulations!** You've built a complete, production-ready system for distributing customized Ollama binaries to your users! 🎉
