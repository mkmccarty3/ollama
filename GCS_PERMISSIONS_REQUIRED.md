# GCS Permissions Required for GitHub Actions

The GitHub Actions workflow for building and uploading Ollama binaries requires specific GCS permissions.

## Current Status

The service account `github-actions-ollama@wikihealthy.iam.gserviceaccount.com` currently has:
- ✅ `storage.objects.create` - Can upload new files
- ❌ `storage.objects.delete` - **MISSING** - Cannot overwrite existing files
- ❌ `storage.objects.list` - **MISSING** - Cannot list bucket contents

## Required Permissions

To enable full functionality, grant these permissions to the service account:

### Option 1: Use Predefined Role (Recommended)
```bash
# Grant "Storage Object Admin" role for both buckets
gsutil iam ch serviceAccount:github-actions-ollama@wikihealthy.iam.gserviceaccount.com:roles/storage.objectAdmin \
  gs://minibase-ollama-binaries-staging

gsutil iam ch serviceAccount:github-actions-ollama@wikihealthy.iam.gserviceaccount.com:roles/storage.objectAdmin \
  gs://minibase-ollama-binaries
```

### Option 2: Grant Specific Permissions
```bash
# Create custom role with specific permissions
gcloud iam roles create OllamaBinaryUploader --project=wikihealthy \
  --permissions=storage.objects.create,storage.objects.delete,storage.objects.get,storage.objects.list

# Grant the custom role
gsutil iam ch serviceAccount:github-actions-ollama@wikihealthy.iam.gserviceaccount.com:projects/wikihealthy/roles/OllamaBinaryUploader \
  gs://minibase-ollama-binaries-staging

gsutil iam ch serviceAccount:github-actions-ollama@wikihealthy.iam.gserviceaccount.com:projects/wikihealthy/roles/OllamaBinaryUploader \
  gs://minibase-ollama-binaries
```

## Current Workaround

The GitHub Actions workflow has been updated to:
1. **Always** upload to versioned directories: `gs://{bucket}/versions/{version}_{timestamp}/`
2. **Attempt** to upload to `latest/` but gracefully handle failures

This ensures binaries are always available, even if the `latest/` update fails.

## Impact

### With Current Permissions (Missing delete/list):
- ✅ Versioned uploads work perfectly
- ⚠️  `latest/` directory cannot be updated after first upload
- ⚠️  Each build creates a new versioned directory but doesn't update the symlink-like `latest/`

### After Fixing Permissions:
- ✅ Versioned uploads work perfectly
- ✅ `latest/` directory gets updated with each build
- ✅ PHP download API can always use `latest/` path
- ✅ Simpler bucket structure

## Next Steps

1. **Immediate**: Grant the required permissions using Option 1 above
2. **Verify**: Trigger a new build and confirm `latest/` updates successfully
3. **Cleanup**: Remove old versioned directories if needed (or keep for rollback)

## Buckets

- **Staging**: `minibase-ollama-binaries-staging`
- **Production**: `minibase-ollama-binaries`

Both buckets need the same permissions.

