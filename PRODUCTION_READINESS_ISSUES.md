# 🚨 Production Readiness Issues

**Critical Review Date**: October 10, 2025  
**Status**: REQUIRES FIXES BEFORE DEPLOYMENT

---

## ❌ CRITICAL ISSUES (Must Fix)

### 1. **API Key Retrieval - Non-Functional**
**File**: `rostra/mediawiki/extensions/ApiKeyAuth/includes/Api/ApiGenerateOllamaBinary.php`  
**Lines**: 88-105

**Problem**:
```php
// We only have the prefix, but for binary generation we need the full key
// This is a limitation - we can't retrieve full keys from database
// Solution: Generate a new key specifically for Ollama or require user to copy it

// For now, return a placeholder that tells user to use their key
return $keyRow->key_prefix . '-FULL-KEY-NEEDED';
```

**Impact**: 🔴 **BROKEN FUNCTIONALITY**
- Binary will be built with placeholder API key: `"mb_abc123-FULL-KEY-NEEDED"`
- User's binary will NOT work - all API requests will fail authentication
- This makes the entire download feature non-functional

**Why This Happened**:
- API keys are stored as hashes in the database (security best practice)
- Only the prefix is stored in plain text
- Full key is only available at creation time
- Binary needs the full key embedded at compile time

**Solutions**:
1. **Option A - Generate New Key (RECOMMENDED)**:
   ```php
   private function getOrCreateOllamaApiKey($user, $orgId): string {
       // Check if user has an "Ollama" API key
       $existing = $this->getOllamaKey($user, $orgId);
       if ($existing) return $existing;
       
       // Generate new API key specifically for Ollama
       $newKey = $this->generateNewApiKey($user, $orgId, 'Ollama CLI');
       return $newKey; // Full key available at creation
   }
   ```

2. **Option B - User Input Modal**:
   - Before download, show modal asking user to copy their API key
   - User pastes key → binary is built with it
   - More manual but doesn't create extra keys

3. **Option C - Store Full Keys (NOT RECOMMENDED)**:
   - Add encrypted column to store full keys
   - Decrypt when needed for binary generation
   - Security risk, violates best practices

**Recommended Fix**: Option A - Auto-generate an "Ollama CLI" API key on first download

---

### 2. **Hardcoded Domain URL**
**File**: `rostra/mediawiki/extensions/ApiKeyAuth/includes/Api/ApiGenerateOllamaBinary.php`  
**Line**: 18

**Problem**:
```php
private const REGISTRY_URL = 'https://yourdomain.com';  // TODO: Make configurable
```

**Impact**: 🟡 **REQUIRES CONFIGURATION**
- Binary will connect to `yourdomain.com` (obviously wrong)
- Must be manually updated before deployment
- No environment-specific configuration (staging vs production)

**Solution**:
```php
private function getRegistryUrl(): string {
    // Check for staging environment
    $isStaging = file_exists('/var/www/html/.staging');
    
    if ($isStaging) {
        return 'https://staging.yourdomain.com';
    }
    
    // Get from MediaWiki config or use server name
    $config = $this->getConfig();
    if ($config->has('OllamaRegistryURL')) {
        return $config->get('OllamaRegistryURL');
    }
    
    // Fallback to current server
    return 'https://' . $_SERVER['HTTP_HOST'];
}
```

Then update in extension.json:
```json
"config": {
    "OllamaRegistryURL": {
        "value": "https://yourproductiondomain.com"
    }
}
```

---

### 3. **Hardcoded Ollama Source Path**
**File**: `rostra/mediawiki/extensions/ApiKeyAuth/includes/Api/ApiGenerateOllamaBinary.php`  
**Line**: 17

**Problem**:
```php
private const OLLAMA_SOURCE_PATH = '/opt/ollama-minibase';
```

**Impact**: 🟡 **DEPLOYMENT DEPENDENCY**
- Requires exact path on server
- No flexibility for different environments
- Will fail with error message if path doesn't exist

**Solution**:
Same as above - make configurable:
```php
private function getOllamaSourcePath(): string {
    $config = $this->getConfig();
    if ($config->has('OllamaSourcePath')) {
        return $config->get('OllamaSourcePath');
    }
    
    // Check common locations
    $paths = [
        '/opt/ollama-minibase',
        '/var/www/ollama-minibase',
        __DIR__ . '/../../../../../ollama-minibase'  // Relative path
    ];
    
    foreach ($paths as $path) {
        if (is_dir($path)) return $path;
    }
    
    throw new Exception('Ollama source not found. Configure OllamaSourcePath.');
}
```

---

## ⚠️ MEDIUM ISSUES (Should Fix)

### 4. **No Build Timeout**
**File**: `rostra/mediawiki/extensions/ApiKeyAuth/includes/Api/ApiGenerateOllamaBinary.php`  
**Lines**: 150-153

**Problem**:
```php
// Execute build (with timeout)  <- Comment says "with timeout" but no timeout!
$output = [];
$returnCode = 0;
exec( $cmd, $output, $returnCode );
```

**Impact**: 🟡 **POTENTIAL HANG**
- If Go build hangs, PHP process will hang forever
- Could tie up PHP-FPM workers
- No way to detect stuck builds

**Solution**:
```php
// Set script timeout
set_time_limit(120); // 2 minutes max

// Use proc_open with timeout
$descriptorspec = [
    0 => ["pipe", "r"],
    1 => ["pipe", "w"],
    2 => ["pipe", "w"]
];

$process = proc_open($cmd, $descriptorspec, $pipes);
if (is_resource($process)) {
    stream_set_blocking($pipes[1], false);
    stream_set_blocking($pipes[2], false);
    
    $startTime = time();
    $output = '';
    
    while (true) {
        $status = proc_get_status($process);
        if (!$status['running']) break;
        
        if (time() - $startTime > 90) { // 90 second timeout
            proc_terminate($process);
            throw new Exception('Build timeout after 90 seconds');
        }
        
        sleep(1);
    }
    
    $returnCode = $status['exitcode'];
    fclose($pipes[0]);
    fclose($pipes[1]);
    fclose($pipes[2]);
    proc_close($process);
}
```

---

### 5. **No Cleanup on Build Failure**
**File**: `rostra/mediawiki/extensions/ApiKeyAuth/includes/Api/ApiGenerateOllamaBinary.php`  
**Lines**: 107-180

**Problem**:
- If build fails after creating temp directory, directory is not cleaned up
- If ZIP creation fails, binary file left in temp directory
- Could accumulate failed builds over time

**Impact**: 🟡 **DISK SPACE LEAK**
- Temp directory will fill up with abandoned builds
- Each failed build leaves ~50-200MB of files

**Solution**:
```php
private function buildBinary( $user, string $apiKey, string $platform, string $arch ): string {
    $tempDir = sys_get_temp_dir() . '/minibase-build-' . $user->getId() . '-' . time();
    $zipPath = null;
    
    try {
        // ... build process ...
        
        return $zipPath;
        
    } catch (Exception $e) {
        // Cleanup on failure
        if (isset($tempDir) && is_dir($tempDir)) {
            $this->recursiveDelete($tempDir);
        }
        if (isset($zipPath) && file_exists($zipPath)) {
            unlink($zipPath);
        }
        throw $e;
    }
}

private function recursiveDelete(string $dir): void {
    if (!is_dir($dir)) return;
    
    $files = array_diff(scandir($dir), ['.', '..']);
    foreach ($files as $file) {
        $path = "$dir/$file";
        is_dir($path) ? $this->recursiveDelete($path) : unlink($path);
    }
    rmdir($dir);
}
```

---

### 6. **Missing GCS Error Handling**
**File**: `rostra/mediawiki/extensions/OllamaRegistry/includes/ApiGetBlob.php`  
**Lines**: 124-175

**Problem**:
- No try-catch around GCS operations
- If GCS client initialization fails, uncaught exception
- Stream errors not handled during download

**Impact**: 🟡 **POOR ERROR MESSAGES**
- Users see generic PHP errors instead of helpful messages
- No logging of GCS failures

**Solution**:
```php
private function streamModelFile( array $model ): void {
    try {
        // Parse GCS URI
        if ( !preg_match( '/^gs:\/\/([^\/]+)\/(.+)$/', $model['gcs_artifact_uri'], $matches ) ) {
            $this->dieWithError( 'Invalid GCS URI format', 'invalid-gcs-uri' );
        }
        
        $bucketName = $matches[1];
        $objectPath = $matches[2];
        
        // Initialize GCS client
        try {
            $credPath = '/var/www/html/scripts/google-cloud-creds.json';
            if ( file_exists( $credPath ) ) {
                $storage = new StorageClient( ['keyFilePath' => $credPath] );
            } else {
                $storage = new StorageClient();
            }
        } catch (Exception $e) {
            wfDebugLog( 'ollamaregistry', 'GCS client init failed: ' . $e->getMessage() );
            $this->dieWithError( 'Storage service unavailable', 'gcs-error' );
        }
        
        // ... rest of function ...
        
    } catch (Exception $e) {
        wfDebugLog( 'ollamaregistry', 'Stream failed: ' . $e->getMessage() );
        $this->dieWithError( 'Download failed. Please try again.', 'stream-error' );
    }
}
```

---

### 7. **No Concurrent Build Protection**
**File**: `rostra/mediawiki/extensions/ApiKeyAuth/includes/Api/ApiGenerateOllamaBinary.php`

**Problem**:
- User could click download button multiple times
- Each click starts a new 60-second build
- No rate limiting or in-progress detection

**Impact**: 🟡 **RESOURCE WASTE**
- Multiple Go compilations running simultaneously
- High CPU usage
- Wasted bandwidth

**Solution**:
```php
private function checkBuildInProgress(int $userId): bool {
    $lockFile = sys_get_temp_dir() . "/minibase-build-lock-{$userId}";
    
    // Check if lock exists and is recent
    if (file_exists($lockFile)) {
        $lockTime = filemtime($lockFile);
        if (time() - $lockTime < 120) { // 2 minute lock
            return true; // Build in progress
        }
        // Stale lock, remove it
        unlink($lockFile);
    }
    
    // Create lock
    touch($lockFile);
    return false;
}

public function execute() {
    // ... auth checks ...
    
    if ($this->checkBuildInProgress($user->getId())) {
        $this->dieWithError('A build is already in progress. Please wait.', 'build-in-progress');
    }
    
    try {
        // ... build process ...
    } finally {
        // Remove lock when done
        $lockFile = sys_get_temp_dir() . "/minibase-build-lock-{$user->getId()}";
        if (file_exists($lockFile)) unlink($lockFile);
    }
}
```

---

## ℹ️ MINOR ISSUES (Nice to Fix)

### 8. **Backfill Script Not Using Maintenance Base Properly**
**File**: `rostra/scripts/backfill_artifact_digests.php`  
**Line**: 23

**Problem**:
```php
require_once __DIR__ . '/../mediawiki/maintenance/Maintenance.php';
```

**Impact**: 🟢 **COSMETIC**
- Should extend Maintenance for proper MediaWiki integration
- Missing some Maintenance helper methods
- Works but not following MediaWiki best practices

**Solution**: Already acceptable for a maintenance script

---

### 9. **JavaScript Download Uses setTimeout Instead of Actual Completion**
**File**: `rostra/mediawiki/extensions/ApiKeyAuth/modules/ext.ApiKeyAuth.apikeys.js`

**Problem**:
```javascript
// Close modal after a delay
setTimeout(function() {
    $('#ollama-platform-modal').hide();
    // ...
}, 2000);
```

**Impact**: 🟢 **UX ISSUE**
- Modal closes after 2 seconds regardless of actual download status
- If download fails, user doesn't know
- If download takes longer, modal closes too early

**Solution**:
- Download endpoint should return JSON status first
- Then redirect to actual binary download
- Or use polling to check build status

---

### 10. **No Model Digest Validation on Download**
**File**: `rostra/mediawiki/extensions/OllamaRegistry/includes/ApiGetBlob.php`

**Problem**:
- Digest parameter is accepted but not validated
- User could request wrong digest
- No verification that requested digest matches model

**Impact**: 🟢 **MINOR SECURITY**
- Potential confusion if mismatched digests
- Ollama client expects digest to match file

**Solution**:
```php
public function execute() {
    // ... existing code ...
    
    $model = $this->validateModelAccess( $modelId, $user );
    
    // Validate digest if provided
    if ($digest && $model['artifact_digest'] !== $digest) {
        wfDebugLog('ollamaregistry', "Digest mismatch: requested={$digest}, actual={$model['artifact_digest']}");
        $this->dieWithError('Digest mismatch', 'invalid-digest');
    }
    
    // ... continue ...
}
```

---

## 📊 SUMMARY

| Severity | Count | Must Fix Before Prod? |
|----------|-------|-----------------------|
| 🔴 Critical | 3 | ✅ YES |
| 🟡 Medium | 4 | ⚠️ Recommended |
| 🟢 Minor | 3 | ❌ Optional |

### Critical Path to Production:

1. **Fix API Key Issue** (#1) - 🔴 BLOCKS ALL FUNCTIONALITY
2. **Fix Hardcoded URLs** (#2, #3) - 🔴 BLOCKS DEPLOYMENT
3. Add build timeout (#4) - 🟡 Prevents hangs
4. Add cleanup on failure (#5) - 🟡 Prevents disk issues
5. Add GCS error handling (#6) - 🟡 Better UX
6. Add concurrent build protection (#7) - 🟡 Prevents resource waste

---

## 🛠️ ESTIMATED FIX TIME

| Issue | Complexity | Time | Priority |
|-------|-----------|------|----------|
| #1 API Key | Medium | 30-45 min | P0 |
| #2 Registry URL | Easy | 15 min | P0 |
| #3 Source Path | Easy | 10 min | P0 |
| #4 Build Timeout | Medium | 20 min | P1 |
| #5 Cleanup | Easy | 15 min | P1 |
| #6 GCS Errors | Easy | 15 min | P1 |
| #7 Concurrency | Easy | 20 min | P1 |

**Total Time**: ~2.5 hours to make production-ready

---

## ✅ WHAT'S ACTUALLY PRODUCTION READY

### These Components Are Good:

1. ✅ **Database Migration** - Fully functional
2. ✅ **OllamaRegistry Extension** - API structure is solid
3. ✅ **Apache Rewrite Rules** - Documentation is complete and accurate
4. ✅ **UI Components** - HTML/CSS/modal structure works
5. ✅ **Model Ownership Logic** - Privacy checks are correct
6. ✅ **GCS Streaming** - Basic streaming logic is sound
7. ✅ **Digest Calculation** - Works correctly in training processor
8. ✅ **Backfill Script** - Functional maintenance script

### What Needs Work:

1. ❌ **Binary Builder** - API key issue makes it non-functional
2. ⚠️ **Configuration** - Hardcoded values need to be configurable
3. ⚠️ **Error Handling** - Needs more robustness
4. ⚠️ **Resource Management** - Needs better cleanup and limits

---

## 🎯 RECOMMENDATION

**DO NOT DEPLOY TO PRODUCTION** until Issue #1 (API Key) is fixed.

The core architecture is solid, but the binary builder will generate non-functional binaries due to the placeholder API key. Users would download a ~50MB file that doesn't work, creating a terrible experience.

**Suggested Approach**:
1. Fix Issue #1 first (critical blocker)
2. Fix Issues #2-3 (quick and required)
3. Test end-to-end with real binary
4. Deploy to staging
5. Add Issues #4-7 as follow-up improvements
6. Deploy to production

---

**Last Updated**: October 10, 2025  
**Reviewed By**: AI Code Reviewer  
**Status**: AWAITING FIXES

