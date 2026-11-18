# Aggregation & Request-Surface Gap Analysis

Scope: functionality monk (`fix/monk-response-format`) still needs so that a client can send
the identical readable request and consume the identical response — **response shape
`outputVersion: '1'` only** (the flat `{count, data, aggs}` body). Derived from monk's own
public source only — no references to other systems, no deployment configuration values, no
document contents, no identifiers.

The use case in scope uses a single aggregation surface: `terms` facets on fields that live
at a child-document (nested) level, requested alongside the normal result fields. All other
aggregation types are out of this scope — see §6.

---

## 1. Query-builder gaps

### 1.1 Terms aggregation on a child-document field (P0)

Monk rejects aggregations on subdocument fields outright
(`AggregationContext.requireFacetField` throws "Aggregations are only supported on root
document fields"). This use case's facet target is a child-document-level field, so this is
the core feature and blocks everything else.

Required Solr translation (both engines need the same domain semantics):

```
<aggName>: {
  type:   "terms",
  field:  <child-level search field>,
  limit:  <size>,
  domain: {
    blockChildren: <child document identifier mask>,
    filter:        ["<nest-path field>:\\/<level path>"]
  }
}
```

Monk already has the raw ingredients from the `nested`/`reverseNested` aggregation work in
`AggregationContext` (`solrNestPathMask`, `solrRootBlockMask`, nest-path resolution, strict
per-material-type resolution with mismatch errors). The work is a new path through the terms
builder that:

1. Resolves the target field's owning subdocument hierarchy (reuse `enterNested` machinery).
2. Emits the `domain.blockChildren` + `domain.filter` wrapping above.
3. Validates the field's facet-type compatibility against the child mapping, not the root.

Elasticsearch side: the equivalent nested-domain wrapping — an ES `nested` aggregation around
the `terms` bucketing, scoped to the same level, so bucket keys are child-field values
counted per parent context.

### 1.2 Uniqueness companion facet under a terms facet (P0)

The child-level term count this use case reports per bucket is not the raw document count
but a distinct-block count: a per-bucket companion facet invoking the block-join unique
function (`uniqueBlock({!v=<doc type mask>})` in the Solr facet body). Monk has no mechanism
to inject a companion facet under a terms facet with a per-bucket domain.

Work:

- Builder: for terms-on-child-document facets, auto-inject the companion facet under the
  terms facet (like the ES `avg` value-count companion from `AvgWeights`, but domain-scoped).
- ES equivalent: whatever the ES destination uses for the same per-bucket distinct-entity
  counting (a `cardinality`/`reverse_nested`/`unique` combination), so keys/counts align.

### 1.3 Backend-side facet sort (P0)

The terms facet in §1.1 sorts its buckets by that companion unique count descending
(`sort: <facetName> desc`) before returning. Monk today sends no `sort` on terms facets.
Even without a client-facing `order` argument, this backend-side default must ship with §1.1,
or merged buckets from the two engines rank differently.

---

## 2. Result-parser gaps

### 2.1 Companion-count preference in bucket parsing (P0)

Monk's default terms parser reads `bucket.count` (Solr `count`, ES `doc_count`). When the
builder injects a §1.2 companion facet, the parser must prefer that facet's value per bucket
(`bucket.<companionName>` when present) over `count`. This is monk's parser change in
`Aggregation.bucketsResult` / `TermsAggregation.parseElasticsearch|parseSolr`, keyed off a
companion facet the builder injected and a marker the client never names directly.

### 2.2 Zero-count bucket suppression (P1)

Add Solr `mincount: 1` (this use case's facet result never carries empty buckets; the ES
`terms` default already excludes them). Consider `refine: true` for deep distributed facets.
Verify `ResponseFormatter`'s post-merge terms re-rank stays idempotent after the drop.

### 2.3 Result parser — currently existing surface

For completeness, monk's aggregation parser surface as implemented today (per agg type):
`Aggregation.bucketsResult` (default `val`/`count` and `key`/`doc_count` dispatch),
`TermsAggregation.parseElasticsearch` (key normalization), `UniqueAggregation`,
`MetricAggregation`, and the doc-count overrides on `Filter`/`Nested`/`ReverseNested`
(`doc_count`/`count`), plus `SubfacetsAggregation.namedCounts`. Items §2.1/§2.2 are the only
parser changes this use case needs; everything else already exists on both engines.

---

## 3. Response-shape (`outputVersion: '1'` flat) — closure items

The flat shape itself is implemented (`ResponseFormatter`); the remaining closures against
the target contract:

### 3.1 Hit-row object shape (P1)

Flat hits are `{field: value}` objects keyed by logical field names. Verify against the
target expectation that multi-valued fields keep their array shape and that key insertion
order matches the requested/resolved field list — where they diverge, either re-key
construction by the requested order or document the difference.

### 3.2 Carrier field (already merged)

The flat contract's per-hit category marker (the mapping's `responseFormat` carrier field)
is already implemented and injected per search-engine hit. No further work; listed so the
audit is explicit.

### 3.3 `matchedQueries` (P2)

Not exercised by this use case (no named queries). Keep monk's behavior; no compatibility
work required here.

### 3.4 `count` semantics (no work)

Flat count `{value, isExact}` with sum-across-backends and exactness rules already matches
the contract. Listed so the audit is explicit.

---

## 4. Request-reader gaps

Monk today accepts `{username, query, fields, size, sort, aggs}`.
Additional contract keys and aliases required for the target client body to parse unchanged:

### 4.1 Request body keys

| Contract | Monk today | Required work |
|---|---|---|
| `resultsRange {start, size}` | `size` only | **P1**: add `start` (per-backend `offset`/ES `from` + merged pagination) |
| `rankBy {field, order, score}` | `sort [{field, order}]` + `_score` token | **P1**: accept the `rankBy` shape (score flag semantics, default order value, uppercase order values) — either map into monk sort or accept both |
| `source` | `fields` | **P1**: alias input (read either key) so the target client body parses unchanged |
| `rerank` | nothing | **P2**: needs a rerank calling layer or explicit deferral |
| `destinationOverride` | catalog routing only | **P2**: explicit backend routing override; low relevance for this use case |
| `outputversion` header | supported (`'1'` flat) | no work — other values explicitly not planned |
| `debugQuery=true` | `/queries/parse` endpoint | acceptable divergence; optionally add a debug flag returning the backend bodies in-band |
| `username` header variant | request-body field | accepted shape difference; optionally also read the header for parity |
| additional request headers | ignored | per-deployment note; no core work |

### 4.2 Readable request structure (the query itself)

The target client sends one condition group per entry, each with a group envelope containing
a readable query tree. Monk's clause model (`QueryNode` leaf/boolean shapes with `bool` tags)
already accepts the envelope; the dialect-level deltas:

```
{
  "id":    <group id>,                      // ✅ same
  "name":  <group name>,                    // ✅ same
  "extra-keys": [],                         // — ignored on monk side today; keep ignoring
  "materialTypes": [ ... ],                 // ✅ same
  "query": {                                // readable tree — see table below
    "field": "",
    "isNot": false,
    "data": [ [ { ...clause... } ] ]        // nested arrays: outer SHOULD, inner MUST
  }
}
```

Dialect deltas to accept (an acceptance layer translating into monk's clause model):

| Readable form | Monk equivalent today | Work |
|---|---|---|
| Nested-array grouping: `data: [[{...}, {...}], [...]]` — outer elements OR, elements inside an inner array AND | boolean nodes with clause `bool` tags | **P1**: translate the array-of-arrays shorthand into monk boolean nodes |
| `isNot: true` on a group or clause | explicit `bool: "mustNot"` tags | **P1**: read `isNot`, convert to mustNot clauses |
| Range clause `{field, data: {from, to}}` (missing side = open) | `{type: range, gte/gt/lte/lt}` | **P1**: map `from`/`to` → `gte`/`lte` (an open side = omitted bound) |
| Dates as `YYYY-MM-DD HH:mm:ss` | ISO-8601 per schema | **P1**: accept the format at the dialect layer (convert at parse) |
| One range per clause is implied (a clause wraps a single range object) | same | ✅ |
| `minimumShould` sugar | `minimumMatch` | **P1**: alias |
| `isScoring` flag (true = scoring) | `filtering` flag (true = non-scoring wrap) | **P1**: map/invert |
| Terms clause: `{field, data: [<v1>, <v2>]}` — list = OR among values | `{type: exact, values: [...]}` = same OR semantics | ✅ shape mapping |
| `bool` keyword values `must` / `mustNot` / `should` | same tags | ✅ |
| `context` tags, `useOperator` sugar | — | audit against real client traffic before implementing |

### 4.3 Query payload families audit (P1)

The readable dialect feeds a query surface wider than monk's current payload set
(`text`/`phrase`/`capsule`, `range`, `exact` numeric/datetime/boolean, `prefix`, `exists`,
`knnFlat`). The readable layer should also accept the same five payload families
(`date`/`number` ranges with from/to, terms-lists, text/capsule/exact/prefix) mapped onto
monk's nodes. Semantic paths, template-based range fan-outs, and duration-style narrow
types are not necessities for this use case — audit actual client traffic before adding.

---

## 5. Implementation order

1. **§1.1 terms on child-document field** — ES+Solr domain wrapping; unblocks everything else.
2. **§1.2 companion unique-count facet + §2.1 parser preference** — as one change (builder
   injects, parser prefers), incl. the ES equivalent.
3. **§1.3 backend-side `sort`** on that facet.
4. **§4.2 readable-clause dialect acceptance layer** + **§4.1 request-key aliases**
   (`start`, `rankBy`, `source`) — completes "identical readable request".
5. **§2.2 Solr `mincount: 1`**.
6. **§3.1 hit-row shape verification** (closure, not new code).
7. **P2** — `rerank`, `destinationOverride`, optional header-parity conveniences.

---

## 6. Explicitly out of scope

Recorded for completeness, not in this project's critical path:

- **`outputVersion: '0'`-style partitioned response** — explicitly not supported; `'1'` only.
- `filters` (dict form with `otherBucketKey`), `bucketScripts`, `bucketSort`, `valueCount`,
  `histogram` — named-bucket, pipeline-arithmetic, and pipeline-sort capabilities used by
  other facets of the product, not by this use case (which exercises `terms` only).
- `terms`/`range` client-facing `order` argument and inline `filters` argument — for this
  use case only the §1.3 backend-side default sort matters.
- Engine-only wrapper-agg fallback lookup — monk's builders don't emit wrappers; stay strict.
- Deprecated alias types; per-endpoint business reshaping of agg output (monk returns the
  normalized agg object and the client reshapes).
