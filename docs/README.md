# LoRaWAN Message Storage — HTTP API Reference

This document describes the HTTP API used to access LoRaWAN uplink messages stored in CouchDB. Messages are collected from Actility ThingPark Network Server and persisted as-is. The API is provided through four pre-built CouchDB views (design documents) and is intended for backend integrations, dashboards, and analytics.

---

## 1. Connection

### Endpoint

```
http://{host}:{port}/{database}
```

Replace `{host}`, `{port}`, and `{database}` with values provided to you. Default CouchDB port is `5984`.

### Authentication

HTTP Basic Auth with a read-only user. Credentials are passed inline in the URL:

```
http://{user}:{password}@{host}:{port}/{database}/...
```

### Network access

The CouchDB endpoint is firewalled. Only allowlisted static IPs can reach it. If your requests time out, your source IP is not on the allowlist — request access from the provider.

### Output

All endpoints return JSON. CouchDB serves `gzip` if you send `Accept-Encoding: gzip`.

---

## 2. The four views

All views live under design documents and are queried via:

```
GET /{database}/_design/{ddoc}/_view/{view}
```

| Design doc   | View          | Answer this question                         | Has reduce |
|--------------|---------------|----------------------------------------------|------------|
| `by_eui`     | `by_eui_time` | What did device X transmit between T1 and T2?| No         |
| `by_lrr`     | `by_lrr_time` | What did gateway X receive between T1 and T2?| No         |
| `by_time`    | `by_time_eui` | What was active between T1 and T2?           | No         |
| `stats`      | `averages`    | What are the long-term signal stats per device? | Yes     |

---

## 3. Common query parameters

These apply to all four views. All values must be **JSON-encoded** and then URL-encoded.

| Parameter        | Type    | Default  | Purpose                                                            |
|------------------|---------|----------|--------------------------------------------------------------------|
| `startkey`       | JSON    | —        | Lower bound of the key range (inclusive).                          |
| `endkey`         | JSON    | —        | Upper bound of the key range (inclusive).                          |
| `limit`          | int     | unbounded| Max rows returned. **Always set this.** Recommended ≤ `1000`.      |
| `descending`     | bool    | `false`  | Reverse the result order. When `true`, **swap** `startkey`/`endkey`.|
| `include_docs`   | bool    | `false`  | Attach the full source document to each row. Expensive.            |
| `update`         | enum    | `true`   | `true` (block until index is fresh), `lazy` (serve now, refresh in background), `false` (never refresh). |
| `stable`         | bool    | `false`  | Stick to one shard replica per query. Useful for paginated reads.  |
| `reduce`         | bool    | `true` if view has reduce | Run the reduce function.                              |
| `group`          | bool    | `false`  | Group reduce output by key.                                        |
| `group_level`    | int     | —        | For composite keys, group at the Nth element.                      |

> **Default freshness recommendation:** use `update=lazy` for dashboards and analytics, `update=true` only when you must see the absolute latest message. With millions of documents and continuous writes, blocking on index refresh can stall the request for seconds.

### Pagination

Do **not** use `skip` on this dataset — it scans linearly.

Use keyset pagination instead:

1. Query with `limit=1000`.
2. Take the `key` and `id` of the **last** row in the response.
3. Re-query with `startkey={last_key}&startkey_docid={last_id}&skip=1&limit=1000`.

Repeat until fewer than `limit` rows are returned.

### Key format

Keys are JSON arrays. `Time` is an ISO 8601 string in **UTC with explicit offset**:

```
2024-01-15T08:30:00.000+00:00
```

Do not mix this format with `Z` notation — they do not sort byte-for-byte the same in CouchDB's collation.

---

## 4. View reference

### 4.1 `by_eui/by_eui_time` — uplinks per device, over time

**Use case:** show a device's recent transmissions, draw signal-quality timelines for a device, export a device's data for a time window.

**Endpoint:**
```
GET /{database}/_design/by_eui/_view/by_eui_time
```

**Key shape:**
```json
["{DevEUI}", "{Time ISO 8601}"]
```

**Value shape** (positional array):

| Index | Field         | Notes                                                           |
|-------|---------------|-----------------------------------------------------------------|
| 0     | `Time`        | ISO 8601 UTC string.                                            |
| 1     | `FCntUp`      | Uplink frame counter.                                           |
| 2     | `FCntDn`      | Downlink frame counter.                                         |
| 3     | `LrrRSSI`     | RSSI at the best gateway, dBm.                                  |
| 4     | `LrrSNR`      | SNR at the best gateway, dB.                                    |
| 5     | `InstantPER`  | Instant packet error rate (0–1).                                |
| 6     | `MeanPER`     | Sliding-window packet error rate (0–1).                         |
| 7     | `TxPower`     | Transmit power, dBm.                                            |
| 8     | `DevLrrCnt`   | Number of gateways that received this uplink.                   |
| 9     | `SpFact`      | Spreading factor (7–12).                                        |
| 10    | `payload_hex` | Raw payload as hex string.                                      |
| 11    | `Lrrid`       | Best gateway ID.                                                |
| 12    | `payload`     | Decoded payload if Actility provided one; otherwise `undefined`.|

**Example — last 24 hours for one device, newest first:**

```http
GET /{database}/_design/by_eui/_view/by_eui_time
  ?startkey=["0080E1150502CAFF","2024-01-16T00:00:00.000+00:00"]
  &endkey=["0080E1150502CAFF","2024-01-15T00:00:00.000+00:00"]
  &descending=true
  &limit=1000
  &update=lazy
```

(URL-encode the brackets, quotes, commas, `+` and `:` in the keys before sending.)

**Response:**
```json
{
  "total_rows": 115610328,
  "offset": 4231,
  "rows": [
    {
      "id": "00025ec7220fa3867863a86e5d000243",
      "key": ["0080E1150502CAFF", "2024-01-15T23:58:12.104+00:00"],
      "value": [
        "2024-01-15T23:58:12.104+00:00", 412, 198,
        -100, 8.75, 0, 0, 16, 3, 7,
        "01034b047700000e4200009b", "1000000D", null
      ]
    }
  ]
}
```

**Caveats:**
- `total_rows` is the total in the view index, **not** the result count.
- Index 12 (`payload`) is often missing — always null-check.
- A wide window (months) for a chatty device can still return tens of thousands of rows. Always paginate.

---

### 4.2 `by_lrr/by_lrr_time` — uplinks per gateway, over time

**Use case:** gateway health diagnostics, coverage analysis, "what did gateway X hear last hour", per-gateway RSSI/SNR distributions.

**Endpoint:**
```
GET /{database}/_design/by_lrr/_view/by_lrr_time
```

**Key shape:**
```json
["{Lrrid}", "{Time ISO 8601}"]
```

Note: `Lrrid` here is the **primary (best) receiving gateway** for that uplink. Each document is emitted once. If the same uplink was heard by 5 gateways, only the primary one appears in this view. The full `Lrrs` array lives in the source document; use `include_docs=true` to retrieve it.

**Value shape** (positional array):

| Index | Field        |
|-------|--------------|
| 0     | `DevEUI`     |
| 1     | `Time`       |
| 2     | `FCntUp`     |
| 3     | `FCntDn`     |
| 4     | `LrrRSSI`    |
| 5     | `LrrSNR`     |
| 6     | `InstantPER` |
| 7     | `MeanPER`    |
| 8     | `TxPower`    |
| 9     | `DevLrrCnt`  |
| 10    | `SpFact`     |

**Example — last hour for one gateway:**

```http
GET /{database}/_design/by_lrr/_view/by_lrr_time
  ?startkey=["1000000D","2024-01-15T23:00:00.000+00:00"]
  &endkey=["1000000D","2024-01-16T00:00:00.000+00:00"]
  &limit=1000
  &update=lazy
```

**Caveats:**
- Counts **only uplinks where this gateway was best**, not every uplink it heard. For full reception data per gateway, fetch the source docs (`include_docs=true`) and inspect `DevEUI_uplink.Lrrs.Lrr[]`.
- A single busy gateway can emit thousands of rows per hour — paginate.

---

### 4.3 `by_time/by_time_eui` — activity in a time window

**Use case:** discover which devices were active in a window, count active devices per interval, audit traffic.

**Endpoint:**
```
GET /{database}/_design/by_time/_view/by_time_eui
```

**Key shape:**
```json
["{Time ISO 8601}", "{DevEUI}"]
```

**Value:** `null`.

Because the value is `null`, this view is for **discovery and counting only**. To inspect a message, either use `include_docs=true` (heavy — adds a doc fetch per row) or take the returned `id` and `GET /{database}/{id}` separately.

**Example — all activity in a 10-minute window:**

```http
GET /{database}/_design/by_time/_view/by_time_eui
  ?startkey=["2024-01-15T10:00:00.000+00:00"]
  &endkey=["2024-01-15T10:10:00.000+00:00"]
  &limit=1000
  &update=lazy
```

**Caveats:**
- This is the most dangerous view to query wide. A 1-hour window across the full fleet can return well into six figures. **Always** combine with a sensible `limit` and paginate.
- `include_docs=true` here multiplies cost — every row triggers a document lookup. Use only when you actually need full payloads and the window is narrow.

---

### 4.4 `stats/averages` — long-term signal stats per device

**Use case:** baseline link quality per device, identify under-performing devices, fleet-wide aggregates.

**Endpoint:**
```
GET /{database}/_design/stats/_view/averages
```

**Map key:** `DevEUI`
**Map value:** `[[LrrRSSI, 1], [LrrSNR, 1], [LrrESP, 1], [SpFact, 1], [DevLrrCnt, 1]]`

The reduce function returns, for each group, an array of `[sum, count]` pairs in this fixed order:

| Index | Metric      |
|-------|-------------|
| 0     | `LrrRSSI`   |
| 1     | `LrrSNR`    |
| 2     | `LrrESP`    |
| 3     | `SpFact`    |
| 4     | `DevLrrCnt` |

**You compute the average on the client:** `mean = sum / count`.

**Per-device averages (most common usage):**
```http
GET /{database}/_design/stats/_view/averages
  ?group=true
  &update=lazy
```

**Single-device average:**
```http
GET /{database}/_design/stats/_view/averages
  ?key="0080E1150502CAFF"
  &group=true
  &update=lazy
```

**Response (single device, `group=true`):**
```json
{
  "rows": [
    {
      "key": "0080E1150502CAFF",
      "value": [
        [-2104150.7, 21043],
        [184126.5, 21043],
        [-2112987.3, 21043],
        [148301, 21043],
        [63129, 21043]
      ]
    }
  ]
}
```

Client computation: `mean_RSSI = -2104150.7 / 21043 ≈ -100.0 dBm`.

**Caveats:**
- `group=true` is what you want 99% of the time. Without it, the reduce collapses the **entire database** into one row — the global mean across every device. Rarely useful.
- `reduce=false` returns raw map output — **do not use it on this view** at scale.
- The map function falls back to `0` when a field is missing. A device that never reports `LrrESP` will skew its own average toward zero. Cross-check with raw queries through `by_eui_time` if you suspect bad data.
- Stats include the device's entire history. There is no time filter on this view. For "last 30 days average", query `by_eui_time` with a date range and aggregate client-side.

---

## 5. Operational guidance

### Choosing freshness vs. speed

| Use case                                    | Recommended setting          |
|---------------------------------------------|------------------------------|
| Real-time alerting on the latest uplink     | `update=true` (default)      |
| Dashboards, charts, analytics               | `update=lazy`                |
| Batch exports, historical analysis          | `update=false`               |

### Estimating cost before you query

- A view query scans a contiguous slice of a B-tree. Range size = response time.
- Narrow time windows are always cheaper than narrow device filters across long time windows.
- `include_docs=true` adds one document fetch per row — multiply your expected row count by ~1ms to estimate added latency.
- `descending=true` does not cost more, but you **must** swap `startkey`/`endkey`.

### Time format — strict

- Use `YYYY-MM-DDTHH:mm:ss.sss+00:00`. UTC offset is explicit, `+00:00`, not `Z`.
- Do not mix formats within a query. Sort order is byte-level lexicographic.

### Connection pooling

CouchDB handles many concurrent connections cheaply. Use a connection pool with keep-alive on the client side. There is no per-IP rate limit configured, but coordinate before running large parallel batch jobs.

### What this API does **not** expose

- Writing, updating, or deleting documents — the user is read-only.
- The `_changes` feed, replication endpoints, or `_design` admin operations.
- Ad-hoc Mango (`_find`) queries — only the four views above are guaranteed to scale on this dataset.

If you need a query pattern not covered by the four views, contact the provider — a new view can be added rather than scanning the database ad-hoc.

---

## 6. Quick reference

```
# Device timeline
GET /{db}/_design/by_eui/_view/by_eui_time
    ?startkey=["{DevEUI}","{T1}"]&endkey=["{DevEUI}","{T2}"]
    &limit=1000&update=lazy

# Gateway timeline
GET /{db}/_design/by_lrr/_view/by_lrr_time
    ?startkey=["{Lrrid}","{T1}"]&endkey=["{Lrrid}","{T2}"]
    &limit=1000&update=lazy

# Time-window activity (discovery, IDs only)
GET /{db}/_design/by_time/_view/by_time_eui
    ?startkey=["{T1}"]&endkey=["{T2}"]
    &limit=1000&update=lazy

# Signal stats — per device
GET /{db}/_design/stats/_view/averages
    ?group=true&update=lazy

# Signal stats — one device
GET /{db}/_design/stats/_view/averages
    ?key="{DevEUI}"&group=true&update=lazy

# Fetch a single message by _id
GET /{db}/{document_id}
```

---

*Document maintained by IOTECH. For access requests, IP allowlist changes, or new view requirements, contact your IOTECH point-of-contact.*
