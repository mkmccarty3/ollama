# Minibase Fork - Setup Documentation

## Overview
This is a fork of Ollama customized for Minibase to support:
- Custom registry URLs (pointing to registry.minibase.ai)
- API key authentication for model downloads
- Integration with Minibase's training and inference pipeline

## Setup Information

### Repository Details
- **Fork Location**: `/Users/codemonkey/Projects/ollama-minibase`
- **Your Fork (Origin)**: `https://github.com/mkmccarty3/ollama.git`
- **Upstream**: `https://github.com/ollama/ollama.git`
- **Working Branch**: `minibase`

### Git Configuration
- **rerere.enabled**: true (automatically reuses conflict resolutions)
- **rerere.autoUpdate**: true (automatically stages resolved conflicts)

## Repository Structure

### Key Directories
- `cmd/` - CLI entry points and command handling
- `server/` - Server-side logic and API handlers
- `api/` - API client code
- `auth/` - Authentication handling
- `model/` - Model management
- `llm/` - LLM runtime and inference
- `runner/` - Model runner implementations
- `convert/` - Model conversion utilities

### Build System
- Go 1.25.2
- CMake for C++ components (llama.cpp integration)
- Main entry point: `main.go`

## Phase 1: Complete ✅

### Completed Steps
1. ✅ Forked ollama/ollama to mkmccarty3/ollama on GitHub
2. ✅ Cloned fork to local machine
3. ✅ Added upstream remote for tracking ollama/ollama
4. ✅ Fetched all upstream branches and tags
5. ✅ Created `minibase` working branch
6. ✅ Enabled git rerere for conflict resolution
7. ✅ Installed Go 1.25.2 via Homebrew

## Next Steps: Phase 2 - Discovery & Code Analysis

### Goals
1. Identify where model pulls/downloads happen
2. Find registry client code
3. Locate URL formation logic
4. Understand authentication flow
5. Map out configuration loading

### Key Files to Investigate
- `server/*.go` - Server endpoints for model operations
- `cmd/*.go` - CLI commands (especially pull/push)
- `api/client.go` - API client implementation
- `auth/auth.go` - Authentication logic
- `model/*.go` - Model management

### Search Queries
```bash
# Find registry-related code
grep -r "registry" --include="*.go" | less

# Find pull/download logic
grep -r "pull\|download" --include="*.go" | less

# Find HTTP client usage
grep -r "http.Client\|NewRequest" --include="*.go" | less

# Find environment variable handling
grep -r "os.Getenv\|LookupEnv" --include="*.go" | less
```

## Syncing with Upstream

### Weekly Update Process
```bash
# In ollama-minibase directory
cd /Users/codemonkey/Projects/ollama-minibase

# Fetch upstream changes
git fetch upstream

# Rebase minibase branch on upstream/main
git checkout minibase
git rebase upstream/main

# If conflicts occur, resolve them and continue
git rebase --continue

# Push to your fork (after testing)
git push origin minibase --force-with-lease
```

## Building the Project

### Quick Build
```bash
cd /Users/codemonkey/Projects/ollama-minibase
go build -o bin/minibase ./main.go
```

### Test the Binary
```bash
./bin/minibase --version
./bin/minibase --help
```

## License
- **Original Project**: MIT License (maintained)
- **Fork**: MIT License
- All original copyright notices and license files preserved

## Notes
- Keep changes minimal and isolated to registry/auth code
- Prefix all commit messages with "minibase:" for easy identification
- Document all modifications in MINIBASE_CHANGES.md
- Avoid modifying inference/runtime code unless absolutely necessary

---
**Last Updated**: October 10, 2025 (Friday)
**Phase**: 1 Complete, Moving to Phase 2

