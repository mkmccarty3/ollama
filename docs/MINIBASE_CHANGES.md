# Minibase Changes - Discovery & Implementation Log

**Last Updated**: October 10, 2025  
**Phase**: 2 - Discovery & Code Analysis

---

## Phase 2: Discovery Results

### 🔍 Registry URL Configuration

**Current Implementation**:
- Default registry URL: `registry.ollama.ai`
- Defined in TWO locations:
  1. `types/model/name.go` line 38:
     ```go
     const defaultHost = "registry.ollama.ai"
     ```
  2. `server/modelpath.go` line 26:
     ```go
     const DefaultRegistry = "registry.ollama.ai"
     ```

**URL Formation**:
- `server/modelpath.go` line 109 - `ModelPath.BaseURL()` method:
  ```go
  func (mp ModelPath) BaseURL() *url.URL {
      return &url.URL{
          Scheme: mp.ProtocolScheme,
          Host:   mp.Registry,
      }
  }
  ```
- `server/modelpath.go` line 40 - `ParseModelPath()` parses model names

**Minibase Strategy**:
- ✅ Add environment variable: `MINIBASE_REGISTRY_URL`
- ✅ Modify both constants to check env var first
- ✅ Fall back to upstream registry if not set

---

### 🚀 Model Pull Logic Flow

**CLI Entry Point**:
- `cmd/cmd.go` - `PullHandler` function
- Creates `api.PullRequest` and calls API endpoint

**API Endpoint**:
- `server/routes.go` line 776 - `PullHandler`:
  ```go
  func (s *Server) PullHandler(c *gin.Context) {
      // ... validation ...
      regOpts := &registryOptions{
          Insecure: req.Insecure,
      }
      if err := PullModel(ctx, name.DisplayShortest(), regOpts, fn); err != nil {
          ch <- gin.H{"error": err.Error()}
      }
  }
  ```

**Core Pull Logic**:
- `server/images.go` line 616 - `PullModel`:
  ```go
  func PullModel(ctx context.Context, name string, regOpts *registryOptions, fn func(api.ProgressResponse)) error {
      mp := ParseModelPath(name)
      // ... pulls manifest ...
      manifest, err = pullModelManifest(ctx, mp, regOpts)
      // ... downloads blobs ...
      downloadBlob(ctx, downloadOpts{...})
  }
  ```

**HTTP Request Layer**:
- `server/images.go` line 815 - `makeRequest`:
  ```go
  func makeRequest(ctx context.Context, method string, requestURL *url.URL, 
                   headers http.Header, body io.Reader, regOpts *registryOptions) (*http.Response, error) {
      // Line 830-835: Add authentication
      if regOpts != nil {
          if regOpts.Token != "" {
              req.Header.Set("Authorization", "Bearer "+regOpts.Token)
          } else if regOpts.Username != "" && regOpts.Password != "" {
              req.SetBasicAuth(regOpts.Username, regOpts.Password)
          }
      }
  }
  ```

**Minibase Strategy**:
- ✅ Intercept `registryOptions` creation in `PullHandler`
- ✅ Populate `Token` field from `MINIBASE_API_KEY` env var
- ✅ Existing code will automatically add Authorization header

---

### 🔐 Authentication Architecture

**registryOptions Struct**:
- Location: `server/images.go` line 46
- Already has auth support built-in:
  ```go
  type registryOptions struct {
      Insecure bool
      Username string  // ✅ Basic auth support
      Password string  // ✅ Basic auth support
      Token    string  // ✅ Bearer token support
      CheckRedirect func(req *http.Request, via []*http.Request) error
  }
  ```

**Authentication Flow**:
1. `makeRequest()` checks `regOpts.Token` (line 830)
2. If Token exists → adds `Authorization: Bearer <token>`
3. If no Token but Username/Password exist → uses Basic Auth
4. Challenge-based auth in `server/auth.go` (for Ollama cloud auth)

**Current Authentication**:
- `server/auth.go` - Ollama-specific challenge-response auth
- Uses Ed25519 signing for Ollama cloud services
- Line 53: `getAuthorizationToken()` - gets token from auth challenge

**Minibase Strategy**:
- ✅ Leverage existing `Token` field in `registryOptions`
- ✅ NO need to modify `makeRequest()` - it already supports Bearer tokens
- ✅ Simply populate `regOpts.Token` from environment variable
- ✅ Keep upstream auth as fallback for public models

---

### ⚙️ Configuration System

**Environment Variable Pattern**:
- Location: `envconfig/config.go`
- Pattern: Create function that reads env var
- Base function: `Var(key string)` - line 327:
  ```go
  func Var(key string) string {
      return strings.Trim(strings.TrimSpace(os.Getenv(key)), "\"'")
  }
  ```

**Existing Environment Variables**:
- `OLLAMA_HOST` - Server host/port
- `OLLAMA_MODELS` - Models directory
- `OLLAMA_KEEP_ALIVE` - Model memory duration
- `OLLAMA_REMOTES` - Allowed remote model hosts
- `OLLAMA_ORIGINS` - CORS origins
- `OLLAMA_DEBUG` - Debug logging
- Plus 20+ more configuration options

**Configuration Helpers**:
- `String(key)` - Returns string function
- `Bool(key)` - Returns bool function
- `Uint(key, default)` - Returns uint function with default
- `BoolWithDefault(key)` - Returns bool with default

**Minibase Strategy**:
- ✅ Add to `envconfig/config.go`:
  ```go
  var (
      MinibaseRegistryURL = String("MINIBASE_REGISTRY_URL")
      MinibaseAPIKey      = String("MINIBASE_API_KEY")
      MinibaseAllowFallback = BoolWithDefault("MINIBASE_ALLOW_FALLBACK")
  )
  ```
- ✅ Update `AsMap()` to include new vars for documentation
- ✅ NO separate config file needed - env vars are standard

---

### 📦 Alternative: Internal Registry Client

**Location**: `server/internal/client/ollama/registry.go`

**Registry Struct** (line 189):
```go
type Registry struct {
    Cache *blob.DiskCache
    UserAgent string
    Key crypto.PrivateKey  // Ed25519 key for auth
    HTTPClient *http.Client
    MaxStreams int
    Mask string  // Name conversion mask
}
```

**Pull Method** (line 464):
```go
func (r *Registry) Pull(ctx context.Context, name string) error {
    // Internal implementation
}
```

**newRequest Method** (line 946):
```go
func (r *Registry) newRequest(ctx context.Context, method, url string, body io.Reader) (*http.Request, error) {
    // Line 959: Adds Bearer token if Key is set
    if r.Key != nil {
        token, err := makeAuthToken(r.Key)
        req.Header.Set("Authorization", "Bearer "+token)
    }
}
```

**Note**: This is a newer, internal registry client. The main server still uses the simpler approach in `server/images.go`. We'll modify the `server/images.go` path as it's more straightforward.

---

## Implementation Plan

### Files to Modify

#### 1. **Add Configuration** (envconfig/config.go)
```go
// Add these new functions (around line 206):
var (
    // ... existing vars ...
    
    // Minibase Configuration
    MinibaseRegistryURL   = String("MINIBASE_REGISTRY_URL")
    MinibaseAPIKey        = String("MINIBASE_API_KEY")
    MinibaseAllowFallback = BoolWithDefault("MINIBASE_ALLOW_FALLBACK")
)
```

Update `AsMap()` (line 270):
```go
func AsMap() map[string]EnvVar {
    ret := map[string]EnvVar{
        // ... existing entries ...
        
        // Minibase
        "MINIBASE_REGISTRY_URL":    {"MINIBASE_REGISTRY_URL", MinibaseRegistryURL(), "Custom registry URL for Minibase (default: registry.ollama.ai)"},
        "MINIBASE_API_KEY":         {"MINIBASE_API_KEY", "***", "API key for Minibase registry authentication"},
        "MINIBASE_ALLOW_FALLBACK":  {"MINIBASE_ALLOW_FALLBACK", MinibaseAllowFallback(true), "Allow fallback to upstream registry for public models"},
    }
    // ...
}
```

#### 2. **Update Registry URL** (server/modelpath.go)
Modify `ParseModelPath()` function (line 40):
```go
func ParseModelPath(name string) ModelPath {
    // Check for Minibase registry override
    defaultRegistry := DefaultRegistry
    if minibaseRegistry := envconfig.MinibaseRegistryURL(); minibaseRegistry != "" {
        defaultRegistry = minibaseRegistry
    }
    
    mp := ModelPath{
        ProtocolScheme: DefaultProtocolScheme,
        Registry:       defaultRegistry,  // Use potentially overridden registry
        Namespace:      DefaultNamespace,
        Repository:     "",
        Tag:            DefaultTag,
    }
    // ... rest of function unchanged ...
}
```

#### 3. **Add Authentication** (server/routes.go)
Modify `PullHandler()` (line 776):
```go
func (s *Server) PullHandler(c *gin.Context) {
    // ... existing validation ...
    
    regOpts := &registryOptions{
        Insecure: req.Insecure,
        Token:    envconfig.MinibaseAPIKey(),  // ← ADD THIS LINE
    }
    
    // ... rest of function unchanged ...
}
```

Also update `PushHandler()` similarly (line 827).

#### 4. **Update Model Name Defaults** (types/model/name.go)
Modify `DefaultName()` function (line 49):
```go
func DefaultName() Name {
    host := defaultHost
    if minibaseHost := envconfig.MinibaseRegistryURL(); minibaseHost != "" {
        host = minibaseHost
    }
    
    return Name{
        Host:      host,  // Use potentially overridden host
        Namespace: defaultNamespace,
        Tag:       defaultTag,
    }
}
```

#### 5. **Update Binary Name** (Optional - Phase 3)
- Update build scripts to output `minibase` instead of `ollama`
- Update version strings and user agent
- Update help text

---

## Environment Variables

### New Minibase Environment Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `MINIBASE_REGISTRY_URL` | string | `registry.ollama.ai` | Custom registry URL (e.g., `registry.minibase.ai`) |
| `MINIBASE_API_KEY` | string | _(empty)_ | Bearer token for registry authentication |
| `MINIBASE_ALLOW_FALLBACK` | bool | `true` | Allow fallback to upstream Ollama registry |

### Usage Examples

```bash
# Pull from Minibase registry with auth
export MINIBASE_REGISTRY_URL="registry.minibase.ai"
export MINIBASE_API_KEY="your-api-key-here"
minibase pull mymodel

# Allow fallback to public Ollama models
export MINIBASE_ALLOW_FALLBACK=true
minibase pull llama2  # Falls back to registry.ollama.ai if not in Minibase

# Disable fallback (Minibase only)
export MINIBASE_ALLOW_FALLBACK=false
```

---

## Testing Strategy

### Unit Tests to Add
1. Test `ParseModelPath()` with `MINIBASE_REGISTRY_URL` set
2. Test `makeRequest()` includes Authorization header when Token is set
3. Test fallback behavior when Minibase registry returns 404

### Integration Tests
1. Mock registry server responding to authenticated requests
2. Test pull from custom registry
3. Test auth failure scenarios
4. Test fallback to upstream

### Manual Testing
```bash
# Build binary
cd /Users/codemonkey/Projects/ollama-minibase
go build -o bin/minibase ./main.go

# Test with environment variables
export MINIBASE_REGISTRY_URL="localhost:5000"
export MINIBASE_API_KEY="test-key"
./bin/minibase pull test-model
```

---

## Minimal Change Philosophy

### Why This Approach is Minimal

1. **Reuses Existing Infrastructure**:
   - `registryOptions.Token` already exists
   - `makeRequest()` already adds Bearer auth
   - Environment variable pattern already established

2. **No Core Logic Changes**:
   - Pull flow unchanged
   - HTTP client unchanged
   - Error handling unchanged

3. **Isolated Changes**:
   - Only 5 files modified
   - ~30 lines of new code
   - All changes in configuration/initialization

4. **Backward Compatible**:
   - If env vars not set → behaves exactly like upstream
   - Existing tests continue to pass
   - No breaking changes

### Upstream Merge Strategy

When syncing with upstream:
- Changes are in distinct sections
- Configuration additions are additive
- Core logic remains untouched
- Conflicts will be minimal and easy to resolve

---

## Next Steps (Phase 3)

1. ✅ Document complete (this file)
2. Create `internal/minibase/` package (optional wrapper)
3. Implement configuration changes
4. Implement registry URL override
5. Implement API key injection
6. Update binary name and branding
7. Build and test
8. Commit with "minibase:" prefix

---

## Files Summary

### Files to Read/Understand ✅
- [x] `types/model/name.go` - Model name parsing
- [x] `server/modelpath.go` - Registry URL formation
- [x] `server/routes.go` - API endpoints
- [x] `server/images.go` - Pull logic and HTTP requests
- [x] `server/auth.go` - Authentication logic
- [x] `envconfig/config.go` - Environment variables
- [x] `cmd/cmd.go` - CLI commands

### Files to Modify 📝
- [ ] `envconfig/config.go` - Add Minibase env vars
- [ ] `server/modelpath.go` - Override default registry
- [ ] `server/routes.go` - Inject API key into regOpts
- [ ] `types/model/name.go` - Override default host
- [ ] Build configuration (later) - Binary name

### Files Created 📄
- [x] `docs/MINIBASE_SETUP.md`
- [x] `docs/MINIBASE_PLAN.md`
- [x] `docs/PHASE_1_COMPLETE.md`
- [x] `docs/MINIBASE_CHANGES.md` (this file)

---

**Discovery Phase Complete**: October 10, 2025  
**Ready for**: Phase 3 - Implementation

