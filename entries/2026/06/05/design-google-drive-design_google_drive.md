# File: design-google-drive/design_google_drive.py

**Date:** 2026-06-05
**Time:** 13:25

## Purpose

This file is a **single-module simulation of Google Drive's backend storage layer**, built as a system design interview reference implementation. It owns the core file storage domain: CRUD operations on files and folders, version history, chunked uploads for large files, permission-based sharing, conflict detection for multi-device sync, quota management, and trash/recovery. Everything runs in-memory — no database, no network — isolating the *design decisions* from infrastructure concerns.

## Key Components

### Data Classes

**`Permission`** — Three-tier enum (`READ < WRITE < ADMIN`) with a companion `PERMISSION_LEVEL` dict that maps each variant to an integer for comparison. This is the authorization primitive used everywhere.

**`FileMetadata`** — The central entity. Represents both files and folders (discriminated by `is_folder`). Notable fields:
- `shared_with: dict[str, Permission]` — per-user ACL stored directly on the file
- `version_vector: dict[str, int]` — maps `device_id → version_number`, used for conflict detection
- `is_deleted` / `deleted_at` — soft-delete markers for trash semantics

**`FileVersion`** — Immutable snapshot of file state at a point in time. Stores both metadata (`content_hash`, `size_bytes`) and the actual `content: bytes`. The version list is the full edit history.

**`ChunkedUpload`** — Tracks an in-progress multipart upload session. Holds individual chunks and checksums until `complete_chunked_upload` assembles them.

### `FileStore` — The Main Class

Single class that acts as the entire storage service. Constructor parameters set system-wide policy: `max_versions` (cap on version history depth, default 100) and `default_quota_bytes` (1 GB per user).

Internal state is organized into parallel dictionaries keyed by `file_id`:
- `self.files` — metadata registry
- `self.content` — current file bytes
- `self.versions` — version history lists
- `self.chunked_uploads` — in-flight upload sessions
- `self.user_quotas` — per-user byte counters
- `self.user_roots` — per-user root folder IDs (lazy-created)

### Key Methods by Category

**Folder ops**: `create_folder`, `list_folder`, `get_path` — standard tree operations. `get_path` walks parent pointers to reconstruct the full `/a/b/c` path.

**File CRUD**: `upload_file`, `download_file`, `update_file`, `delete_file`, `move_file`, `rename_file`. Every mutating operation checks permissions and updates quotas.

**Versioning**: `get_versions`, `restore_version` — `restore_version` doesn't do a raw rollback; it calls `update_file` with the old content, creating a *new* version that happens to match an old one. This preserves the audit trail.

**Chunked upload**: `init_chunked_upload` → `upload_chunk` (repeated) → `complete_chunked_upload`. Quota is reserved upfront at init, then released-and-reaccounted at completion. `abort_chunked_upload` releases the reservation.

**Sharing**: `share`, `revoke`, `get_shared_with` — ACL management. Only owner or `ADMIN`-level users can share.

**Conflict detection**: `detect_conflict` uses version vectors to distinguish true conflicts (concurrent edits from different devices) from simple staleness. `resolve_conflict` supports two strategies: `latest_wins` (no-op, server version stands) and `keep_both` (creates a `(conflict)` copy).

**Trash**: `list_trash`, `restore_from_trash`, `empty_trash` — soft-delete with time-based permanent deletion (default 30 days, computed via `auto_days * 86400` seconds).

**Search**: Linear scan over all files with substring name matching and optional MIME type filter. Only returns files the user can read.

## Patterns

**Soft delete** — Files are never immediately removed from `self.files`. Deletion sets `is_deleted=True` and records `deleted_at`. Permanent removal only happens in `empty_trash` based on age.

**Lazy root creation** — `_get_or_create_root` ensures every user has a root folder, created on first access. This avoids upfront user registration.

**Quota reservation** — Chunked uploads reserve quota at init time, not completion. This prevents a user from starting multiple uploads that collectively exceed their quota. The reservation is released then re-accounted when the upload finalizes.

**Permission inheritance** — `_check_permission` walks up the folder tree via `parent_folder_id` pointers. If any ancestor grants sufficient access, the check passes. This models Google Drive's "shared folder" semantics.

**Version vectors for conflict detection** — Rather than simple version counters, each file tracks which version each device last wrote. A conflict is detected when another device has a version newer than what the requesting device last saw — distinguishing concurrent edits from sequential ones.

**Content-addressed integrity** — SHA-256 checksums are computed on upload/update and stored in both `FileMetadata.checksum` and `FileVersion.content_hash`.

## Dependencies

**Imports**: All stdlib — `hashlib` (checksums), `math` (ceiling division for chunks), `uuid` (ID generation), `dataclasses`, `enum`, `typing`. No external dependencies.

**Imported by**: `test_design_google_drive.py` — the test suite is the sole consumer.

## Flow

A typical file lifecycle:

1. **Upload**: `upload_file` → quota check → generate UUID → compute SHA-256 → create `FileMetadata` → store content → create initial `FileVersion` (v1)
2. **Edit**: `update_file` → permission check → quota delta adjustment → new checksum → bump `current_version` → update version vector for device → append `FileVersion` → prune if over `max_versions`
3. **Sync conflict**: Device B calls `detect_conflict(file_id, "deviceB", base_version=1)` → file is at v3, device A wrote v2 and v3 → `version_vector["deviceA"] > version_vector.get("deviceB", 0)` → returns `True`
4. **Delete**: `delete_file` → soft-delete → if folder, cascade to all descendants via BFS in `_get_descendants`
5. **Trash cleanup**: `empty_trash` → filter by `deleted_at <= cutoff` → hard-delete from all stores → release quota

For chunked uploads: `init_chunked_upload` reserves quota → N calls to `upload_chunk` storing individual chunks → `complete_chunked_upload` assembles chunks in order, releases reservation, delegates to `upload_file` for the actual storage, then cleans up the upload session.

## Invariants

- **Quota accounting is balanced**: every byte added via `upload_file` or `update_file` is tracked; every byte removed via `empty_trash` or `abort_chunked_upload` is released. Chunked uploads reserve upfront and release-then-reaccount on completion.
- **Permission checks guard all reads and writes**: `download_file`, `update_file`, `delete_file`, `move_file`, `rename_file`, `list_folder`, and `search` all call `_check_permission`.
- **Soft-delete cascades to descendants**: deleting a folder marks all children (recursively) as deleted.
- **Version list is bounded**: `update_file` prunes to `max_versions`, keeping only the most recent entries.
- **Chunked uploads must have all chunks**: `complete_chunked_upload` checks `range(total_chunks)` and raises on any missing index.
- **Owners bypass permission checks**: `_check_permission` returns immediately if `meta.owner_id == user_id`.

## Error Handling

Three exception types are used consistently:

| Exception | Raised when |
|-----------|------------|
| `FileNotFoundError` | File ID doesn't exist or file is soft-deleted |
| `PermissionError` | User lacks required permission level |
| `ValueError` | Quota exceeded, missing chunks, unknown conflict strategy, or version not found |

Errors are raised immediately — nothing is swallowed. Callers (the test suite) are expected to handle them. There's no retry logic or partial-failure recovery. `abort_chunked_upload` and `delete_file` are the only methods that return `bool` success/failure instead of raising.

## Topics to Explore

- [file] `design-google-drive/test_design_google_drive.py` — See how conflict detection, chunked uploads, and permission inheritance are exercised in tests
- [function] `design-google-drive/design_google_drive.py:detect_conflict` — The version vector logic is the most interview-relevant part; trace through multi-device edit scenarios
- [file] `design-google-drive/plan.md` — Understand the design decisions and tradeoffs considered before implementation
- [general] `version-vector-vs-lamport-clock` — Why version vectors were chosen over simpler conflict detection; how they scale with device count
- [file] `s3-object-storage/s3_object_storage.py` — Compare the object storage design: different chunking, deduplication, and metadata strategies for a related problem

## Beliefs

- `gdrive-permission-inheritance` — Permission checks walk up the folder tree via parent pointers; access granted on any ancestor grants access to all descendants
- `gdrive-quota-reservation` — Chunked uploads reserve full quota at init and release-then-reaccount at completion, preventing over-commitment
- `gdrive-version-vector-conflict` — Conflict detection uses per-device version vectors, not simple version counters; a conflict requires a *different* device to have written a version the requesting device hasn't seen
- `gdrive-soft-delete-cascade` — Deleting a folder soft-deletes all descendants via BFS traversal; permanent deletion only occurs in `empty_trash` after a time-based cutoff
- `gdrive-restore-creates-new-version` — `restore_version` delegates to `update_file`, so restoring to an old version creates a new version entry rather than rolling back the version counter

