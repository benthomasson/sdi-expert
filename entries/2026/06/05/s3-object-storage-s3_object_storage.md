# File: s3-object-storage/s3_object_storage.py

**Date:** 2026-06-05
**Time:** 14:01

# `s3-object-storage/s3_object_storage.py`

## Purpose

This file is a self-contained, in-memory simulation of Amazon S3's core API surface. It exists as a system design interview reference implementation — not a production storage engine, but a demonstration that the candidate understands S3's data model, versioning semantics, multipart upload protocol, presigned URL access control, and bucket policy evaluation. It owns the entire storage plane: bucket lifecycle, object CRUD, listing with prefix/delimiter pagination, copy, multipart upload assembly, presigned token generation/validation, storage class transitions, and policy checks.

## Key Components

### Data Classes

**`ObjectVersion`** — The atomic unit of storage. Every write produces one of these, whether the bucket is versioned or not. Notable fields:
- `version_id`: `str` (UUID when versioned, `None` when not)
- `is_delete_marker`: soft-delete sentinel used only in versioned buckets
- `etag`: MD5 hex digest of `data`, matching real S3's ETag behavior for non-multipart objects

**`Bucket`** — Container with four parallel dictionaries:
- `objects`: `key → list[ObjectVersion]` — the version chain
- `multipart_uploads`: `upload_id → {part_number: bytes}` — in-flight parts
- `multipart_meta`: `upload_id → key` — maps uploads back to their target key
- `policies`: flat list of policy dicts (evaluated in order)

### `ObjectStorage` Class

The public API class. Two class-level constants:
- `_BUCKET_RE`: enforces S3's bucket naming rules (lowercase alphanumeric + hyphens, 3–63 chars)
- `_SECRET`: per-process HMAC key for presigned URL signing (generated once at class load, not per instance)

**Bucket operations**: `create_bucket`, `delete_bucket`, `list_buckets`

**Object CRUD**: `put_object`, `get_object`, `delete_object`, `head_object`, `copy_object`

**Listing**: `list_objects` (prefix/delimiter/pagination), `list_object_versions`

**Multipart**: `initiate_multipart` → `upload_part` (×N) → `complete_multipart` | `abort_multipart`

**Access control**: `generate_presigned_url`, `access_presigned`, `set_bucket_policy`, `check_bucket_policy`

**Lifecycle**: `transition_storage_class`

## Patterns

**Version chain as append-only list.** Each key maps to a `list[ObjectVersion]`. The latest version is always `versions[-1]`. In versioned buckets, `put_object` appends; in unversioned buckets, it replaces the entire list with a single-element list. This is the central design decision — it makes `_latest_version` trivial (check the tail) and version listing a flat iteration.

**Delete markers as sentinel versions.** Deleting a versioned object doesn't remove data — it appends a zero-byte `ObjectVersion` with `is_delete_marker=True`. `_latest_version` returns `None` when the tail is a delete marker, making the object appear gone to callers while preserving history. This mirrors real S3 semantics exactly.

**Token-based presigned URLs.** Rather than generating actual URLs, the implementation uses HMAC-SHA256 tokens stored in `_presigned` dict. `generate_presigned_url` creates the token; `access_presigned` validates it and delegates to `get_object`. This is a clean abstraction — the signing scheme is realistic while avoiding HTTP concerns.

**Flat policy evaluation with short-circuit.** `check_bucket_policy` iterates policies in insertion order, returning on the first match. DENY wins over ALLOW when it matches first. Default is allow-all (no policies = open bucket, unmatched principal/action = allow).

## Dependencies

**Imports** — all stdlib:
- `hashlib` / `hmac`: ETag computation (MD5) and presigned URL signing (SHA256)
- `re`: bucket name validation
- `time`: timestamps for `last_modified`, `created`, presigned expiry
- `uuid`: version IDs, upload IDs, presigned URL nonce
- `dataclasses`: `ObjectVersion` and `Bucket` structure

**Imported by**: `test_s3_object_storage.py` — the test suite is the only consumer.

## Flow

### Write path
`put_object` → validate bucket exists → coerce `str` to `bytes` → compute MD5 ETag → generate version ID (UUID if versioned, else `None`) → create `ObjectVersion` → append to version list (versioned) or replace list (unversioned) → return `{version_id, etag, size}`.

### Read path
`get_object` → find bucket → if `version_id` specified, linear scan the version list; otherwise call `_latest_version` which checks `versions[-1]` and rejects delete markers → return dict via `_version_to_dict(include_data=True)`.

### Delete path (versioned)
`delete_object` → append a delete marker `ObjectVersion` to the version list → return `{deleted, delete_marker, version_id}`. The data is still there; only `_latest_version` changes behavior.

### Multipart upload
`initiate_multipart` → allocate `upload_id`, create empty parts dict → caller uploads parts in any order via `upload_part` → `complete_multipart` pops the parts dict, sorts by part number, concatenates bytes, delegates to `put_object` for storage. Abort cleans up both dicts.

### Listing with prefix/delimiter
`list_objects` collects all keys matching the prefix, sorts them, then for each key: if a delimiter is present and appears in the key's suffix after the prefix, the key is folded into a common prefix (simulating S3's "virtual directory" behavior) and skipped. Otherwise it's added to the results list up to `max_keys`. Pagination uses the last returned key as `continuation_token`.

## Invariants

1. **Bucket names** must match `^[a-z0-9][a-z0-9-]{1,61}[a-z0-9]$` — enforced at `create_bucket`.
2. **Object keys** must be ≤1024 characters — enforced at `put_object`.
3. **Bucket deletion** requires the bucket to be logically empty: no live objects, and in versioned buckets, no non-marker versions at all.
4. **Version lists are never empty while the key exists** in `bucket.objects`. If a non-versioned delete removes the only version, the key is deleted from the dict entirely.
5. **Multipart uploads must have at least one part** — `complete_multipart` raises `ValueError` on empty parts.
6. **Parts are assembled in sorted part-number order** regardless of upload order.
7. **Presigned tokens expire** — `access_presigned` checks `expires_at` against current time.
8. **`_SECRET` is class-level, not instance-level** — all `ObjectStorage` instances in the same process share the same signing key, meaning presigned tokens from one instance are theoretically valid on another (though `_presigned` dict is instance-scoped, so this doesn't actually work cross-instance).

## Error Handling

The module uses a straightforward exception strategy with no custom exception types:

- **`KeyError`**: missing bucket (`_require_bucket`), missing object, missing upload ID, missing presigned token target
- **`ValueError`**: invalid bucket name, bucket not empty on delete, key too long, no parts in multipart, invalid/expired presigned URL, unknown presigned operation
- No errors are swallowed. Every failure path raises immediately.
- `get_object` and `head_object` are the exceptions — they return `None` for missing objects rather than raising, which matches S3's HTTP 404 semantics (not-found is a valid response, not an error).

## Topics to Explore

- [file] `s3-object-storage/test_s3_object_storage.py` — Reveals edge cases the implementation handles and exposes the expected API contract through test scenarios
- [function] `s3-object-storage/s3_object_storage.py:list_objects` — The prefix/delimiter/pagination interaction is the most complex logic in the file; worth tracing with concrete examples to verify the common-prefix folding is correct
- [function] `s3-object-storage/s3_object_storage.py:check_bucket_policy` — The policy evaluation order and default-allow semantics differ from real AWS IAM (which defaults to deny); worth examining whether tests cover DENY-before-ALLOW ordering
- [general] `versioning-delete-marker-lifecycle` — How delete markers interact with `list_objects`, `get_object`, and `delete_bucket` to understand the full soft-delete lifecycle
- [file] `s3-object-storage/plan.md` — The design plan likely documents which S3 features were intentionally scoped in or out

## Beliefs

- `s3-unversioned-put-replaces` — In unversioned buckets, `put_object` replaces the entire version list with a single-element list, making previous data unrecoverable
- `s3-delete-marker-hides-not-removes` — Deleting a versioned object appends a delete marker; all prior versions remain accessible by explicit `version_id`
- `s3-multipart-sorts-by-part-number` — `complete_multipart` assembles parts by sorting on part number, so upload order is irrelevant
- `s3-presigned-secret-is-class-level` — `_SECRET` is generated once per class load (`uuid4().hex`), not per instance, so it's shared across all `ObjectStorage` instances in the same process
- `s3-policy-default-allow` — `check_bucket_policy` returns `True` (allow) when no policies exist or when no policy matches the principal/action pair — opposite of AWS IAM's default-deny

