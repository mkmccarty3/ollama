# Staging/Production Environment Setup

**Last Updated**: October 10, 2025  
**Status**: ✅ FULLY COMPATIBLE

---

## Overview

The OllamaRegistry extension now fully supports both **staging** and **production** environments, following the same patterns used throughout the Minibase codebase.

---

## Environment Detection

### Detection Method
All extensions use the same staging detection pattern:

```php
$isStaging = file_exists('/var/www/html/.staging');
```

- **Staging**: File `/var/www/html/.staging` exists
- **Production**: File `/var/www/html/.staging` does NOT exist

### Logging Convention
Consistent emoji logging across all extensions:

```php
if ($isStaging) {
    wfDebugLog('extension', "🎭 Operation on STAGING environment");
} else {
    wfDebugLog('extension', "🚀 Operation on PRODUCTION environment");
}
```

- 🎭 = Staging
- 🚀 = Production

---

## GCS Bucket Translation

### Bucket Naming Convention

| Environment | Bucket Name |
|-------------|-------------|
| **Production** | `minibase-trained-models` |
| **Staging** | `minibase-trained-models-staging` |

### Cross-Environment Access Pattern

Models may be stored in one environment's bucket but accessed from another environment. The system automatically translates bucket names:

```php
// Handle bucket translation for cross-environment access
$isStaging = file_exists('/var/www/html/.staging');
$expectedBucket = $isStaging ? 'minibase-trained-models-staging' : 'minibase-trained-models';

if ($bucketName !== $expectedBucket) {
    if ($bucketName === 'minibase-trained-models-staging' && !$isStaging) {
        $bucketName = 'minibase-trained-models'; // staging->production
    } elseif ($bucketName === 'minibase-trained-models' && $isStaging) {
        $bucketName = 'minibase-trained-models-staging'; // production->staging
    }
}
```

**This pattern is used in:**
- `ModelPackaging/includes/ApiDownloadPackage.php`
- `OllamaRegistry/includes/ApiGetBlob.php`

---

## Registry URL Detection

### Auto-Detection from HTTP_HOST

The registry URL is automatically determined based on the server's hostname:

```php
private function getRegistryUrl(): string {
    // Check for staging environment
    $isStaging = file_exists('/var/www/html/.staging');
    
    // Get from MediaWiki config (if set)
    $config = $this->getConfig();
    if ($config->has('OllamaRegistryURL')) {
        return $config->get('OllamaRegistryURL');
    }
    
    // Construct from server name (works for both staging and production)
    if (isset($_SERVER['HTTP_HOST'])) {
        $protocol = (!empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off') ? 'https' : 'http';
        return $protocol . '://' . $_SERVER['HTTP_HOST'];
    }
    
    // Fallback
    return $isStaging ? 'https://staging.minibase.ai' : 'https://minibase.ai';
}
```

**How it works:**
- **Production**: `$_SERVER['HTTP_HOST']` = `minibase.ai` → Registry URL: `https://minibase.ai`
- **Staging**: `$_SERVER['HTTP_HOST']` = `staging.minibase.ai` → Registry URL: `https://staging.minibase.ai`

**Priority:**
1. `$wgOllamaRegistryURL` config (if set in `LocalSettings.php`)
2. Auto-detect from `HTTP_HOST`
3. Hardcoded fallback based on staging flag

---

## OllamaRegistry Environment Support

### ApiGetBlob.php ✅
**Purpose**: Stream GGUF model files from GCS

**Staging Support:**
- ✅ Environment detection logging
- ✅ GCS bucket translation
- ✅ Cross-environment model access

```php
// Added environment logging
$isStaging = file_exists('/var/www/html/.staging');
if ($isStaging) {
    wfDebugLog('ollamaregistry', "🎭 Ollama download on STAGING environment");
} else {
    wfDebugLog('ollamaregistry', "🚀 Ollama download on PRODUCTION environment");
}

// Added bucket translation
$isStaging = file_exists('/var/www/html/.staging');
$expectedBucket = $isStaging ? 'minibase-trained-models-staging' : 'minibase-trained-models';
// ... translation logic
```

### ApiGenerateOllamaBinary.php ✅
**Purpose**: Generate custom Ollama binaries with embedded API keys

**Staging Support:**
- ✅ Environment detection logging
- ✅ Registry URL auto-detection
- ✅ Configurable via `$wgOllamaRegistryURL`

```php
// Added environment logging
$isStaging = file_exists('/var/www/html/.staging');
if ($isStaging) {
    wfDebugLog('ollamabinary', "🎭 Ollama binary generation on STAGING environment");
} else {
    wfDebugLog('ollamabinary', "🚀 Ollama binary generation on PRODUCTION environment");
}

// Registry URL embeds correct environment URL
$registryUrl = $this->getRegistryUrl(); // Auto-detects staging vs production
```

### ApiListModels.php ✅
**Purpose**: List models accessible to user (Ollama format)

**Staging Support:**
- ✅ Environment detection logging
- ⚠️ No bucket translation needed (queries DB only)

```php
// Added environment logging
$isStaging = file_exists('/var/www/html/.staging');
if ($isStaging) {
    wfDebugLog('ollamaregistry', "🎭 Ollama list models on STAGING environment");
} else {
    wfDebugLog('ollamaregistry', "🚀 Ollama list models on PRODUCTION environment");
}
```

### ApiGetManifest.php ✅
**Purpose**: Return OCI-style manifests for models

**Staging Support:**
- ✅ Environment detection logging
- ⚠️ No bucket translation needed (queries DB only)

```php
// Added environment logging
$isStaging = file_exists('/var/www/html/.staging');
if ($isStaging) {
    wfDebugLog('ollamaregistry', "🎭 Ollama manifest request on STAGING environment");
} else {
    wfDebugLog('ollamaregistry', "🚀 Ollama manifest request on PRODUCTION environment");
}
```

---

## Configuration

### LocalSettings.php (Optional)

You can override the auto-detection with explicit configuration:

```php
// Force specific registry URL (optional)
$wgOllamaRegistryURL = 'https://custom.domain.com';

// Force specific Ollama source path (optional)
$wgOllamaSourcePath = '/custom/path/to/ollama-minibase';
```

**Typical setup:**
- **Production**: No config needed (uses `HTTP_HOST`)
- **Staging**: No config needed (uses `HTTP_HOST`)

---

## Testing

### Verify Staging Detection

**On staging server:**
```bash
ls -la /var/www/html/.staging
# Should show the file exists
```

**On production server:**
```bash
ls -la /var/www/html/.staging
# Should show "No such file or directory"
```

### Verify Bucket Access

**Check logs for bucket translation:**
```bash
tail -f /var/log/mediawiki/debug.log | grep -i ollama
```

**Expected log entries:**
```
[ollamaregistry] 🎭 Ollama download on STAGING environment
[ollamaregistry] Checking environment and bucket translation...
[ollamaregistry] Environment: STAGING
[ollamaregistry] Expected bucket: minibase-trained-models-staging, original bucket: minibase-trained-models
[ollamaregistry] Bucket mismatch - performing translation
[ollamaregistry] Translated production->staging
[ollamaregistry] Final bucket to use: minibase-trained-models-staging
```

### Verify Binary Generation

**Test registry URL embedding:**

1. Generate binary on staging:
   ```bash
   # Should embed: https://staging.minibase.ai
   ```

2. Generate binary on production:
   ```bash
   # Should embed: https://minibase.ai
   ```

3. Check embedded values in logs:
   ```bash
   tail -f /var/log/mediawiki/debug.log | grep "Building binary with registry"
   ```

---

## Comparison with Other Extensions

### ModelPackaging
**Pattern**: ✅ Same
- Uses `/var/www/html/.staging` detection
- Translates `minibase-trained-models` ↔ `minibase-trained-models-staging`
- Logs with 🎭/🚀 emojis

### FileUploads
**Pattern**: ✅ Same
- Uses `/var/www/html/.staging` detection
- Uses `getStagingAwareBucketName()` helper
- Logs with 🎭/🚀 emojis
- Bucket: `minibase-upload-files-staging` vs `minibase-upload-files`

### ModelTraining
**Pattern**: ✅ Same
- Uses `/var/www/html/.staging` detection
- Passes `IS_STAGING` environment variable to Vertex AI jobs
- Uses staging-aware bucket names for configs

### Marketplace/Onboarding
**Pattern**: ✅ Same
- Uses `/var/www/html/.staging` detection
- Loads environment-specific config arrays
- Logs environment with 🌍 emoji

**OllamaRegistry now follows ALL these patterns consistently.**

---

## Deployment Checklist

### Staging Deployment
- [ ] Ensure `/var/www/html/.staging` file exists
- [ ] Verify `minibase-trained-models-staging` bucket exists
- [ ] Test model downloads from staging
- [ ] Test binary generation embeds `staging.minibase.ai`
- [ ] Check logs show 🎭 emoji

### Production Deployment
- [ ] Ensure `/var/www/html/.staging` file does NOT exist
- [ ] Verify `minibase-trained-models` bucket exists
- [ ] Test model downloads from production
- [ ] Test binary generation embeds `minibase.ai`
- [ ] Check logs show 🚀 emoji
- [ ] Test cross-environment model access (if needed)

---

## Troubleshooting

### Issue: Wrong bucket being accessed

**Symptoms:**
- 404 errors when downloading models
- "Model file not found in storage" errors

**Diagnosis:**
```bash
# Check environment detection
ls /var/www/html/.staging

# Check logs
tail -f /var/log/mediawiki/debug.log | grep "Final bucket to use"
```

**Solution:**
- Verify `/var/www/html/.staging` file presence matches expected environment
- Check bucket translation logs
- Verify GCS buckets exist and have correct names

### Issue: Wrong registry URL in binary

**Symptoms:**
- Downloaded binary connects to wrong environment
- "Registry not found" errors from Ollama client

**Diagnosis:**
```bash
# Check what URL was embedded
tail -f /var/log/mediawiki/debug.log | grep "Building binary with registry"
```

**Solution:**
- Verify `HTTP_HOST` is set correctly
- Check `$wgOllamaRegistryURL` in `LocalSettings.php` if overridden
- Verify `/var/www/html/.staging` file presence

### Issue: Models not visible in staging

**Symptoms:**
- `ollama_listModels` returns empty list
- Models exist in production but not in staging

**Diagnosis:**
- Check database - models may not be in staging DB
- Check `training_models` table for org_id/user_id

**Solution:**
- This is expected - staging and production have separate databases
- Create test models in staging environment
- Or: Copy production models to staging (if testing cross-env access)

---

## Summary

✅ **All OllamaRegistry APIs support both staging and production**
✅ **Auto-detection works via `HTTP_HOST`**
✅ **GCS bucket translation handles cross-environment access**
✅ **Logging follows project-wide conventions**
✅ **Patterns match existing extensions (ModelPackaging, FileUploads, etc.)**

**No manual configuration required in most cases.**

The system automatically detects the environment and adjusts accordingly.

