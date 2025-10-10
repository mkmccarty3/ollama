# Ollama API Implementation Details

## ApiGetBlob.php - What Gets Downloaded?

### Current Implementation ✅ CORRECT

**ApiGetBlob.php** streams **only the raw GGUF file** from GCS. This is the correct behavior for Ollama.

### Why This Is Correct

Ollama's architecture works differently than your ModelPackaging system:

1. **ModelPackaging** (your existing system):
   - Downloads GGUF file + inference.lock.json + model_info.json
   - Packages with custom inference server
   - Uses your proprietary config format

2. **Ollama** (OCI distribution spec):
   - Downloads manifest (metadata about layers)
   - Downloads blobs (actual files referenced in manifest)
   - Ollama handles its own configuration internally

### What Ollama Actually Needs

When a user runs `minibase pull model-123`, here's what happens:

```
1. GET /v2/library/model-123/manifests/latest
   → Returns manifest with:
     - Config layer (optional)
     - Model layer (GGUF file) with digest + size
     
2. GET /v2/library/model-123/blobs/sha256:abc123...
   → Streams the GGUF file
   
3. Ollama saves to ~/.minibase/models/
4. Ollama reads GGUF metadata directly from file
5. No separate config files needed
```

### GGUF Files Are Self-Describing

GGUF format includes metadata inside the file:
- Model architecture
- Tokenizer configuration
- Parameter count
- Context length
- Template information

Ollama reads this metadata directly from the GGUF file using `ggml.Decode()`.

### What's Missing (Optional Enhancement)

For a more complete Ollama implementation, you could add **additional layers** to the manifest:

```php
// In ApiGetManifest.php, you could return:
'layers' => [
    [
        'mediaType' => 'application/vnd.ollama.image.model',
        'digest' => $model['artifact_digest'],
        'size' => $model['artifact_size_mb'] * 1024 * 1024
    ],
    // OPTIONAL: Template layer
    [
        'mediaType' => 'application/vnd.ollama.image.template',
        'digest' => 'sha256:template_hash',
        'size' => 512
    ],
    // OPTIONAL: System prompt layer
    [
        'mediaType' => 'application/vnd.ollama.image.system',
        'digest' => 'sha256:system_hash',
        'size' => 256
    ]
]
```

Then `ApiGetBlob.php` would need to handle these different layer types. But for **minimum viable implementation**, just the GGUF file is sufficient.

### Comparison: Your ModelPackaging vs Ollama

| Feature | ModelPackaging | Ollama |
|---------|---------------|--------|
| Model File | ✅ GGUF | ✅ GGUF |
| Config File | ✅ inference.lock.json | ❌ Not needed (reads from GGUF) |
| Model Info | ✅ model_info.json | ❌ Not needed |
| Inference Server | ✅ Custom Python server | ✅ Built into Ollama |
| Template | ✅ In inference.lock.json | ⚠️ Optional Ollama layer OR reads from GGUF |
| System Prompt | ✅ In inference.lock.json | ⚠️ Optional Ollama layer |

### Conclusion

**Your ApiGetBlob.php is implemented correctly.** It just streams the GGUF file, which is all Ollama needs for basic functionality.

The manifest in ApiGetManifest.php tells Ollama:
- What file to download (digest)
- How big it is (size)
- What type it is (mediaType)

The blob endpoint (ApiGetBlob.php) streams that file.

Ollama does the rest internally.

### If You Want Full Compatibility

To match Ollama's full feature set, you would need to:

1. Store template/system prompt separately (or extract from model metadata)
2. Add them as additional layers in manifest
3. Update ApiGetBlob to serve different layer types based on mediaType
4. Calculate separate digests for each layer

But this is **NOT required** for a working implementation. Just the GGUF file works fine.

---

**Status**: ✅ ApiGetBlob.php is production-ready as-is.

