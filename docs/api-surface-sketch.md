# API & Contract Surface Sketch (forward-looking)

> This is a design sketch, not an implementation. Nothing described here
> exists in code yet — it exists so `lafiya-web` and `lafiya-contract` are
> built against a shared plan instead of diverging. Update this doc, don't
> silently deviate from it, if the real implementation differs.

## `lafiya-web` API (sketch)

| Method | Path | Auth | Purpose |
| --- | --- | --- | --- |
| `POST` | `/api/profile` | Patient session | Create/update the full profile |
| `GET` | `/api/profile/me` | Patient session | Read own full profile |
| `PATCH` | `/api/profile/me/visibility` | Patient session | Toggle per-field public visibility |
| `GET` | `/e/{public_id}` | None | Public emergency page (renders emergency subset + verified indicator) |
| `POST` | `/api/attestations` | Attester session | Submit an attestation for a record hash |
| `GET` | `/api/attestations/{public_id}` | None | Check current attestation status for a record |
| `GET` | `/api/admin/attestations` | Admin session | List attestations across records with filtering and pagination |
| `GET` | `/api/profile/visits` | Patient session | List facility visit history for authenticated patient |

## Reusable List Endpoint Contract (Pagination, Filtering, Sorting)

All current and future list-shaped endpoints in `lafiya-web` must conform to this single, standardized contract. Ad-hoc query parameters or custom response wrappers are prohibited.

### 1. Pagination Specification & Rationale

Lafiya standardizes on **opaque cursor-based pagination** rather than numeric offset/page pagination.

#### Rationale

1. **Immunity to Record Drift**: Emergency healthcare data, audit trails, and attestation logs are continuously appended. Under offset pagination (`?page=2&limit=20`), new records inserted at the head cause items to shift across page boundaries, resulting in duplicate or skipped records for the client. Cursor pagination anchors each page to a deterministic record position.
2. **Predictable Query Latency (`O(1)` seeks)**: Offset queries (`OFFSET 10000 LIMIT 20`) require the database engine to scan and discard 10,000 index entries. Cursor pagination translates to index seek conditions (`WHERE (created_at, id) < (cursor_timestamp, cursor_id)`), ensuring steady-state sub-millisecond query performance regardless of dataset depth.
3. **Resilience on Constrained Mobile Networks**: Community health workers and emergency responders frequently operate over intermittent 2G/EDGE networks. Opaque cursors allow mobile clients to reliably resume synchronization sessions from their last checkpoint without re-requesting prior data.

#### Standard Query Parameters

| Parameter | Type | Required | Default | Bounds | Description |
| --- | --- | --- | --- | --- | --- |
| `limit` | integer | No | `20` | `1` to `100` | Maximum number of items to return in the current page. |
| `cursor` | string | No | null | Base64-encoded | Opaque pointer denoting the boundary of the previous page. Omit to fetch the initial page. |
| `direction` | string | No | `forward` | `forward`, `backward` | Paging navigation direction relative to the cursor. |

### 2. Filtering Query-Parameter Conventions

Filtering query parameters must be explicit, composable, and use `snake_case` matching field names defined in [`data-model.md`](data-model.md).

- **Exact Field Matching**: Pass the field name and desired value directly:
  - `?status=verified`
  - `?attester=GABC1234...`
- **Range & Temporal Comparison**: Suffix parameter names with standard operator tokens:
  - `_gte` (greater than or equal): `?created_after_gte=2026-01-01T00:00:00Z` or `?timestamp_gte=1700000000`
  - `_lte` (less than or equal): `?timestamp_lte=1709999999`
- **Multi-Value (Set Membership)**: Comma-delimited list of accepted enum variants:
  - `?blood_group=O+,A+,B+`
  - `?genotype=AA,AS`
- **Boolean Filters**: Strict lowercase literals:
  - `?has_attestation=true`
- **Unrecognized Parameters**: Endpoints must return an explicit `400 Bad Request` if an unsupported filter parameter is supplied, rather than silently ignoring it.

### 3. Sorting Query-Parameter Conventions

- **Syntax**: `?sort=<field>` for ascending order, `?sort=-<field>` for descending order.
- **Multi-Column Sorting**: Comma-separated list ordered by precedence:
  - `?sort=-created_at,id`
- **Default Sort**: List endpoints default to newest-first order (`?sort=-created_at`).
- **Deterministic Tie-Breaking**: When sorting on non-unique columns (such as `timestamp`, `status`, or `name`), the implementation must append the entity's unique primary identifier (`id` or `public_id`) as the final sort tie-breaker to ensure stable cursor navigation.

### 4. Canonical Response Envelopes

To ensure programmatic client predictability across all services, list responses wrap results in a standard top-level envelope.

#### Success Envelope (Paginated)

```json
{
  "data": [
    {
      "id": "att_01HX4A2...",
      "record_hash": "0x4b7c...",
      "attester": "GABCTEST...",
      "timestamp": 1709999999,
      "status": "verified"
    }
  ],
  "pagination": {
    "limit": 20,
    "has_more": true,
    "next_cursor": "ZXlKMWNtbHphVzVoYldGMGFXOXVJanBi...",
    "prev_cursor": null,
    "total_count": null
  }
}
```

- `data`: Array containing the requested entity representations. Empty array `[]` when no items match.
- `pagination.limit`: Number of items evaluated for this page.
- `pagination.has_more`: Boolean flag indicating if another page of records exists in the navigation direction.
- `pagination.next_cursor`: Base64 cursor pointing to the start of the next page. `null` if `has_more` is `false`.
- `pagination.prev_cursor`: Base64 cursor to traverse backwards, or `null` if at the beginning.
- `pagination.total_count`: Integer representing total matching records if requested via `?include_total=true`; otherwise `null` to avoid expensive database table counts on large datasets.

#### Canonical Error Envelope

List endpoints must conform to the unified platform error envelope on validation or server errors:

```json
{
  "error": {
    "code": "INVALID_PAGINATION_CURSOR",
    "message": "The provided cursor string is invalid or expired.",
    "details": {
      "field": "cursor",
      "reason": "malformed_base64_payload"
    }
  }
}
```

Standard error codes for list operations:

- `INVALID_PAGINATION_CURSOR`: Malformed, corrupted, or unsupported cursor structure.
- `INVALID_LIMIT`: Limit value is outside the `1..100` allowable range.
- `INVALID_FILTER`: Filter parameter contains an unsupported key or malformed type.
- `INVALID_SORT`: Sort key is not indexed or permitted for ordering.

## `lafiya-contract` Soroban interface (sketch)

```rust
pub fn attest(env: Env, attester: Address, record_hash: BytesN<32>) -> u64; // returns ledger timestamp
pub fn get_attestation(env: Env, record_hash: BytesN<32>) -> Option<AttestationRecord>;
pub fn add_attester(env: Env, governance: Address, attester: Address);
pub fn remove_attester(env: Env, governance: Address, attester: Address);
```

This mirrors the
[attestation record schema](data-model.md#attestation-record-on-chain-soroban).
Any change to function names or argument order here is a shared-contract
change per the root README.
