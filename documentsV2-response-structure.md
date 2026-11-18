# `POST /documentsV2` — Response Structure

Request header `outputVersion` selects the response shape:

- `'0'` (server default when the header is missing) → **legacy split shape** — see below
- `'2'` (send this) → **flat shape** — see below

On success this response header is also returned:

    documents-destination: <destination>   (the destination string)

Destination is auto-derived from the condition groups' `materialTypes`:
either the search engine or a collection database (Solr-based).

## Flat shape (`outputVersion: '2'`)

Same body on both destinations:

```json
{
  "count": { "value": 4210, "isExact": false },
  "data": [ { "transactionId": "...", "startTime": "..." } ],
  "aggs": { "toolIds": { "buckets": [ { "key": "1779979290", "value": 79881 } ] } }
}
```

- `data` hits are pre-zipped `{field: value}` objects — no `source` array
- on the search engine each hit carries one extra field: the **carrier field**
  (a per-hit category marker; in xabs it is `communicationType`); on collection
  databases that field is never injected. Read the field name from your own
  RFO mapping, not from this doc
- nested output fields are `^`-prefixed per level (`^` one level, `^^` two) —
  the prefixes are part of the wire format, don't strip them
- don't assume every hit carries every requested field — hits without nested
  docs have their child fields absent
- `count`:
  - search engine: `isExact = relation === 'eq' && value <= 10000`;
    `count` itself is `undefined` if the middleware returned no count
  - collection databases: `isExact = !responseHeader.partialResults`
- `aggs` is at top level on both destinations

## Legacy split shape (`outputVersion: '0'`, server default)

On the search engine the core auto-injects the carrier field and buckets every hit
by its value into **three response keys** (`bucket1`/`bucket2`/`bucket3`, named by
your RFO mapping — in xabs these are the category responses). An unmapped value
aborts the request (500). All four top-level keys are always present, even when
empty:

```json
{
  "bucket1": { "count": "2", "data": [ [["..."], ["..."]] ], "source": ["transactionId", "startTime"] },
  "bucket2": { "count": "0", "data": [], "source": ["transactionId", "startTime"] },
  "bucket3": { "count": "0", "data": [], "source": ["transactionId", "startTime"] },
  "aggs":    { "toolIds": { "buckets": [ { "key": "1779979290", "value": 79881 } ] } }
}
```

- rows are value arrays: zip `data[i][j]` against the bucket's `source[j]`
- `count` is a **string**; when the total is inexact, all three buckets get
  `'>' + total` (the same total for all three)
- the carrier field is removed from rows and `source` unless the client listed
  it in its own requested fields
- `aggs` is at top level, shared by the three buckets

On collection databases there is no split: the whole result sits under **one key**
chosen by destination — `aggs` inside that key, exact string `count`, no
carrier-field injection:

```json
{ "bucket1": { "count": "27606328", "data": [ [["..."]] ], "aggs": {}, "source": ["transactionId"] } }
```

destination → key: a fixed lookup table — in xabs five collections of one family →
`bucket1`, the first family's single collection → `bucket2`, the second family's
collection → `bucket3`, and the combined two-family collection → a fourth key,
`bucket4`. Two other reachable destinations have no mapping — such a response comes
back under the literal key `"undefined"` (server-side defect in xabs; do not treat it
as a contract key). Your repo defines its own table in the RFO mapping.

## Key naming — resolution through the RFO mapping

The response key names above (`bucket1`…`bucket4`) are **logical placeholders,
not wire literals** — never hardcode them in parser code. Each deployment names
them its own way (what xabs writes as `bucket1` another deployment may call
`bookResponse`); the actual names come from a response-mapping section in the
consuming repo's RFO (root-fields-object YAML, hot-reloaded on change):

- `responseKeys` — destination → response key (collection-DB branch above)
- `communicationTypeBuckets` — carrier-field value → bucket (engine split above)
- `splitKeys` — bucket → response key (the three engine keys)

Parser code then stays generic — it buckets and names keys purely by lookup —
and each deployment names its response keys side-by-side with the rest of its
field mapping.
