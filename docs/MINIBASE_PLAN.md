# Minibase Fork Implementation Plan

## Project Goal
Create a maintained fork of Ollama that:
1. Supports custom registry URLs (registry.minibase.ai)
2. Implements API key authentication for model pulls
3. Maintains compatibility with upstream Ollama
4. Integrates with Minibase's model training/publishing pipeline
5. Vendors into the rostra monorepo for deployment

## Phase Overview

### ✅ Phase 1: Fork & Initial Setup (COMPLETE)
- [x] Fork ollama/ollama on GitHub
- [x] Clone fork locally
- [x] Add upstream remote
- [x] Create minibase branch
- [x] Configure git rerere
- [x] Install Go 1.25.2

### 🔄 Phase 2: Discovery & Code Analysis (IN PROGRESS)
**Goal**: Understand how Ollama handles model pulls and registry communication

**Tasks**:
- [ ] Identify model pull/download code paths
- [ ] Find registry client implementation
- [ ] Locate URL formation logic
- [ ] Understand current authentication (if any)
- [ ] Map configuration loading system
- [ ] Document findings in MINIBASE_CHANGES.md

**Key Questions**:
1. Where does Ollama construct registry URLs?
2. How are models downloaded (HTTP client usage)?
3. What configuration files/env vars exist?
4. How are headers added to HTTP requests?
5. Is there any existing auth mechanism?

### ⏳ Phase 3: Implement Minibase Changes
**Goal**: Add minimal, isolated code for custom registry + auth

**Tasks**:
- [ ] Create `internal/minibase/config.go` for configuration
- [ ] Create `internal/minibase/registry.go` for registry client
- [ ] Add environment variable support:
  - `MINIBASE_REGISTRY_URL`
  - `MINIBASE_API_KEY`
  - `MINIBASE_ALLOW_FALLBACK`
- [ ] Integrate config into existing pull logic
- [ ] Add auth header injection
- [ ] Update binary name to `minibase` in build
- [ ] Add branding updates (README, version strings)

**Design Principles**:
- Keep changes minimal and isolated
- Use wrapper/adapter pattern to avoid modifying core code
- Make upstream merges as painless as possible
- All commits prefixed with "minibase:"

### ⏳ Phase 4: Testing & Build
**Tasks**:
- [ ] Build binary locally
- [ ] Test with mock registry endpoints
- [ ] Verify auth headers are sent correctly
- [ ] Test fallback to upstream registry
- [ ] Document configuration options

### ⏳ Phase 5: Vendor into Rostra Monorepo
**Goal**: Make fork available to the Minibase project

**Tasks**:
- [ ] Add fork as git subtree to rostra
  ```bash
  cd /Users/codemonkey/Projects/rostra
  git remote add ollama-fork https://github.com/mkmccarty3/ollama.git
  git subtree add --prefix third_party/ollama ollama-fork minibase --squash
  ```
- [ ] Create build script: `rostra/scripts/build_minibase.py`
- [ ] Test building from monorepo
- [ ] Document usage in rostra docs

### ⏳ Phase 6: Upstream Sync Automation
**Tasks**:
- [ ] Create sync script in fork repo
- [ ] Document sync process
- [ ] Set up weekly reminder for syncing
- [ ] Test conflict resolution workflow

### ⏳ Phase 7: Integration with Minibase Services
**Tasks**:
- [ ] Update training pipeline to use minibase binary
- [ ] Set environment variables in training scripts
- [ ] Create model publishing script
- [ ] Test end-to-end flow: train → publish → pull

### ⏳ Phase 8: Distribution Setup
**Tasks**:
- [ ] Create multi-platform build script
- [ ] Set up release pipeline
- [ ] Package binaries (macOS, Linux, Windows)
- [ ] Create installation documentation

### ⏳ Phase 9: Registry Backend
**Tasks**:
- [ ] Choose registry approach (OCI vs custom)
- [ ] Implement auth gateway
- [ ] Set up blob storage (GCS/S3)
- [ ] Create manifest generation from trained models
- [ ] Test model publication and retrieval

## Commit Message Convention
All Minibase-specific commits should be prefixed:
```
minibase: <description>

Examples:
- minibase: Add configuration loading for custom registry
- minibase: Implement auth header injection for model pulls
- minibase: Update binary name and branding
```

## File Organization

### In Fork Repository
```
ollama-minibase/
├── internal/minibase/        # NEW: Our custom code
│   ├── config.go             # Configuration loading
│   └── registry.go           # Registry client wrapper
├── docs/
│   ├── MINIBASE_SETUP.md     # This file
│   ├── MINIBASE_PLAN.md      # Implementation plan
│   └── MINIBASE_CHANGES.md   # Detailed change log
└── [existing Ollama structure]
```

### In Rostra Monorepo (After Phase 5)
```
rostra/
├── third_party/ollama/       # Vendored via git subtree
├── scripts/
│   ├── build_minibase.py     # Build script
│   └── publish_model_to_registry.py  # Model publishing
└── bin/
    └── minibase              # Built binary
```

## Development Workflow

### Daily Development
1. Work in `/Users/codemonkey/Projects/ollama-minibase`
2. Open in Cursor for editing
3. Make changes in `internal/minibase/` primarily
4. Test locally with `go build`
5. Commit with "minibase:" prefix

### Updating Monorepo
```bash
cd /Users/codemonkey/Projects/rostra
git subtree pull --prefix third_party/ollama ollama-fork minibase --squash
python scripts/build_minibase.py
```

### Syncing with Upstream
```bash
cd /Users/codemonkey/Projects/ollama-minibase
git fetch upstream
git rebase upstream/main
# Resolve conflicts if any
git push origin minibase --force-with-lease
```

## Success Criteria

### Phase 2-4 Complete When:
- [ ] Can build minibase binary
- [ ] Binary accepts MINIBASE_REGISTRY_URL
- [ ] Binary adds Authorization header with MINIBASE_API_KEY
- [ ] Can pull models from custom registry

### Phase 5 Complete When:
- [ ] Fork vendored in rostra monorepo
- [ ] Can build from monorepo
- [ ] Build script works reliably

### Phase 9 Complete When:
- [ ] Can train model in Minibase
- [ ] Can publish model to registry
- [ ] Can pull model with minibase CLI
- [ ] Can run inference on pulled model

## Resources
- **Ollama GitHub**: https://github.com/ollama/ollama
- **Your Fork**: https://github.com/mkmccarty3/ollama
- **Go Documentation**: https://go.dev/doc/
- **OCI Distribution Spec**: https://github.com/opencontainers/distribution-spec

---
**Last Updated**: October 10, 2025
**Current Phase**: Phase 2 (Discovery & Code Analysis)

