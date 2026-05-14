# Plan: Fix HTTP manifest loader gzip misdecode (#158)

## Problem

When loading a `manifest.json` over HTTP(S) via the `file` reference type, `load_from_http()` blindly trusts the `Content-Encoding: gzip` response header to decide whether to decompress. Python's `requests` library **automatically decompresses gzip responses** but **leaves the `Content-Encoding` header in place**. This causes double-decompression: `requests` decompresses the body, then our code sees the stale header and tries to decompress the already-plain JSON → `OSError: Not a gzipped file`.

Repro:

```
Content-Encoding: gzip   # Set by GitHub CDN
response.content:        # Already decompressed by requests
  b'{"metadata": ...'    # Plain JSON, NOT gzip magic bytes
```

## Root Cause

`dbt_loom/manifests.py`, `load_from_http()`, lines ~170-175:

```python
if (
    config.path.path.endswith(".gz")
    or response.headers.get("Content-Encoding") == "gzip"
):
    with gzip.GzipFile(fileobj=BytesIO(response.content)) as gz_file:
        return json.load(gz_file)
```

The `Content-Encoding` header check is unreliable because `requests` strips the compression but not the header.

## Fix Strategy

**Replace header-based detection with gzip magic byte inspection.** The gzip format always starts with the 2-byte magic number `\x1f\x8b`. Check the actual bytes of `response.content` instead of trusting headers.

### Step 1: Update `load_from_http` in `dbt_loom/manifests.py`

Replace the compression detection logic:

```python
# Before:
if (
    config.path.path.endswith(".gz")
    or response.headers.get("Content-Encoding") == "gzip"
):
    with gzip.GzipFile(fileobj=BytesIO(response.content)) as gz_file:
        return json.load(gz_file)

# After:
if _is_gzipped(response.content):
    with gzip.GzipFile(fileobj=BytesIO(response.content)) as gz_file:
        return json.load(gz_file)
```

Add a helper:

```python
_GZIP_MAGIC = b"\x1f\x8b"

@staticmethod
def _is_gzipped(data: bytes) -> bool:
    """Check if data starts with gzip magic bytes."""
    return data[:2] == _GZIP_MAGIC
```

This correctly handles:

- **Plain JSON from GitHub raw URLs** — `requests` auto-decompresses, content is plain JSON → magic bytes don't match → uses `response.json()` ✅
- **Actually gzipped files** (e.g., S3-presigned `.gz` URLs that don't go through `requests` auto-decompress) — magic bytes match → decompresses ✅
- **`.gz` files served with proper `Content-Encoding`** — `requests` auto-decompresses them, so content is plain → falls through to `response.json()` ✅

**Note:** The `.gz` extension check is removed because it's redundant — if the file is actually gzipped, magic bytes catch it; if `requests` already decompressed it, magic bytes correctly say "not gzipped."

### Step 2: Add unit tests in `tests/test_manifest_loaders.py`

Add three new tests using `responses` library (or `pytest-httpserver` if already available) to mock HTTP responses:

1. **`test_load_from_http_plain_json_with_gzip_header`** — Server returns `Content-Encoding: gzip` header but plain JSON body (the exact bug scenario). Should parse JSON successfully without gzip decompression.

2. **`test_load_from_http_actually_gzipped_content`** — Server returns actual gzip-compressed bytes. Should decompress and parse JSON.

3. **`test_load_from_http_plain_json_no_encoding_header`** — Server returns plain JSON with no `Content-Encoding`. Should parse normally (regression test).

Check whether `responses` or `pytest-httpserver` is already a dev dependency; if not, add `responses` (lightweight, already commonly used with `requests` testing).

### Step 3: Verify existing tests still pass

Run the full test suite to ensure no regressions:

```bash
pytest tests/test_manifest_loaders.py -v
```

The existing `test_load_from_remote_pass` test hits a real S3 URL — it should continue to work since the magic-byte check is strictly more correct than the header check.

## Files Modified

| File                                       | Change                                                                               |
| ------------------------------------------ | ------------------------------------------------------------------------------------ |
| `dbt_loom/manifests.py`                    | Replace header-based gzip detection with magic-byte inspection in `load_from_http()` |
| `tests/test_manifest_loaders.py`           | Add 3 new unit tests for HTTP gzip edge cases                                        |
| `pyproject.toml` or `requirements-dev.txt` | Add `responses` dev dependency (if not present)                                      |

## Risks & Mitigations

- **Risk:** Some edge case where the server truly sends gzipped content AND `requests` doesn't auto-decompress (e.g., `stream=True` with certain configurations). **Mitigation:** Magic-byte check handles this correctly — if bytes are actually gzip, we decompress.
- **Risk:** The existing `test_load_from_remote_pass` test depends on a real external URL. **Mitigation:** This test is already flaky by nature; the fix doesn't change its behavior for correctly-encoded responses.

## Out of Scope

- Supporting streaming decompression for very large manifests (the current code already loads full content into memory)
- Adding explicit `Accept-Encoding` header control
- The `test_projects/customer_success` integration test (that's the reporter's local repro; the unit tests cover the logic)
