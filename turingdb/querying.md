---
name: turingdb-querying
description: Use when reading data from TuringDB v3 — MATCH/OPTIONAL MATCH patterns, variable-length and quantified paths, WHERE, WITH, aggregation, UNWIND, lists, CASE, EXISTS/COUNT/CALL subqueries, UNION, built-in functions, LOAD CSV, and result types. Does not cover writes (see writing.md) or shortest path / vector search (see algorithms.md).
---

# TuringDB: Querying

Read queries run directly against the current graph state — no change workflow needed:

```python
df = client.query("MATCH (n:Person) RETURN n.name, n.age")
```

**Before writing queries, know these limits:**
- **No parameters.** `$name` is a parse error and `query()` takes only a string, so build the query text with inlined literals and escape quotes yourself.
- **One statement per request.** `;` chaining is rejected (a single trailing `;` is fine).
- **Comments:** `//` line comments work. SQL-style `--` comments and `/* */` comments do not.

## MATCH

```cypher
MATCH (n) RETURN count(n)
MATCH (n:Person) RETURN n.name, n.age
MATCH (n:Person {name: 'Alice', age: 30}) RETURN n.name            // inline property map
MATCH (n:Person:Employee) RETURN n.name                           // has BOTH labels
MATCH (a:Person)-[:KNOWS]->(b:Person) RETURN a.name, b.name
MATCH (a)-[e:KNOWS {since: 2010}]->(b) RETURN a.name, e.since, b.name
MATCH (a)<-[e]-(b) RETURN a.name, b.name                          // reverse direction
MATCH (a)-[e]-(b) RETURN a.name, b.name                           // undirected (also --)
MATCH (a)-[r:KNOWS|WORKS_AT]->(b) RETURN a.name, type(r), b.name  // edge type alternatives
MATCH (a:Person WHERE a.age > 30)-[r:KNOWS WHERE r.since > 2015]->(b) RETURN b.name  // inline WHERE (variable must be named)
MATCH (a {name: 'Alice'}) MATCH (a)-[:KNOWS]->(b) RETURN b.name   // multiple MATCH clauses
MATCH (n:Person) ORDER BY n.age LIMIT 2 RETURN n.name             // TuringDB extension: ORDER BY/SKIP/LIMIT on MATCH
```

- **Node label alternatives** (`(n:A|B)`, `(n:!A)`, `(n:A&B)`) are **not** supported. Write `WHERE n:A OR n:B` / `WHERE NOT n:A`.
- **Unknown labels and properties don't error.** A label that doesn't exist matches nothing, and a misspelled property silently reads as null.

### OPTIONAL MATCH

```cypher
MATCH (a:Person) OPTIONAL MATCH (a)-[:WORKS_AT]->(c:Company) RETURN a.name, c.name   // c.name null if no match
MATCH (a:Person) OPTIONAL MATCH (a)-[:KNOWS]->(b) RETURN a.name, count(b)            // 0 for no match
```

The WHERE of an OPTIONAL MATCH belongs to the optional part, so rows are kept, as in Neo4j.

**Gotcha:** an unmatched optional **node** returned bare (`RETURN c`) shows `18446744073709551615` (UINT64_MAX) rather than null. Properties (`c.name`) and `id(c)` are correctly null.

### Addressing nodes by native node ID (the fast path)

Every node has a native node ID: the integer you get from `RETURN n` or `id(n)`. **Whenever you already know which node(s) you want, address them by comparing the node variable itself with the native ID, written as a literal in the query: `WHERE n = <id>`.** The planner turns that into a constant scan of exactly those nodes (`const_scan_nodes`). Don't look them up again through a user-level key property such as `n.pid`, `n.uuid` or `n.doc_id`.

```cypher
MATCH (n) WHERE n = 4 RETURN n.name                                   // direct seek
MATCH (n)-[:KNOWS]->(m) WHERE n = 4 RETURN m.name                     // seek, then expand
UNWIND [4, 17, 42] AS x MATCH (n) WHERE n = x RETURN n.name           // batch: folded into one constant scan
UNWIND [4, 17, 42] AS x MATCH (n) WHERE n = x SET n.flag = true       // batch write by ID (inside a change)
```

Measured on a 1M-node graph (server-side time):

| Lookup | Time |
|---|---|
| `WHERE n = <id>` | **0.16 ms** (constant scan) |
| `WHERE id(n) = <id>` | 1.4 ms (full node scan; grows with graph size) |
| `UNWIND [<1,000 literal IDs>] AS x` + `WHERE n = x` | **1.4 ms total (≈1.4 µs per node)** |
| `UNWIND` keys + `MATCH (n:P {pid: x})` | ≈1.1 ms **per node**, index or not (≈800× slower) |

**Rules:**
- **Compare the variable, not the function.** `WHERE n = 123` seeks, but `WHERE id(n) = 123` scans every node. Use `id(n)` only to *return* the ID.
- **Batch with `UNWIND [literal IDs] AS x MATCH (n) WHERE n = x`, not `IN`.** `WHERE n IN [..]` and `WHERE id(n) IN [..]` scan every node.
- **The batch form is fast only in one exact shape.** There is no per-row seek: the planner rewrites the UNWIND into `n = 4 OR n = 17 OR …` at plan time, and that only works when:
  - the list is a **literal written into the query text** (build it in Python: `f"UNWIND {ids} AS x …"`), not a variable, `collect(...)`, `range(...)` or a list property;
  - its **elements are distinct** integers, so dedupe in Python first; one repeated ID makes it fall back to a full scan;
  - `x` is used **only** in `n = x`. Return `id(n)` instead of `x`, because `RETURN x, n.name` falls back to a full scan + cross product.

  Check with `EXPLAIN`: the plan should show `const_scan_nodes`, not `scan_nodes`. A label (`MATCH (n:P) WHERE n = x`) and `SET` are fine.
- **Get the IDs once, reuse them.** Fetch them with whatever filter you need (`MATCH (n:Person {email: 'a@x.org'}) RETURN id(n)`, or `RETURN n`), keep them in Python, and address those nodes by ID from then on.
  - **Key lookups:** a single key lookup with a constant (`{pid: 7}`) is fine. It's per-row key lookups (`UNWIND keys … {pid: x}`, LOAD CSV joins) that are slow.
- **Never look up nodes by IDs computed per row, and never two nodes from one row.** Forms like `WHERE a = p[0] AND b = p[1]`, `a = ss[i]`, or two MATCHes on two values from one row are not rewritten at all. They run as a **cross product of two full node scans**, which can exhaust server memory on large graphs.
  - To pair specific nodes, put both IDs in as literals, one pair per query: `MATCH (a) WHERE a = 5 MATCH (b) WHERE b = 7`.
  - For bulk edges, see `LOAD PARQUET` in `writing.md`.
- **Vector search results:** `VECTOR SEARCH … YIELD ids` binds `ids` directly as nodes. Use them as is (`ids.title`, `MATCH (ids)-[...]->(...)`). Re-matching with `MATCH (n:Label) WHERE n = ids`, or joining on a property, scans the label. `ids` is a runtime column, and only literal IDs become a constant scan.

**Stability.** Native IDs of existing nodes are stable across queries, change submits and server restarts, so it is safe to cache them for a working session. Two exceptions:
- **New nodes:** nodes created in an open change get their final IDs at `COMMIT` / `CHANGE SUBMIT`. Read them after that.
- **Compaction:** `MERGE_DATAPARTS` **renumbers** all nodes. Re-fetch any cached IDs, and rebuild vector indexes keyed by node IDs, afterwards.

Keep a business key property as well, for durable identity outside TuringDB. `elementId()` does not exist. For edges, `MATCH ()-[r]->() WHERE id(r) = 0 RETURN startNode(r), endNode(r)` works.

## Variable-length paths

Both the TuringDB/GQL postfix quantifier and the Neo4j bracket form work:

| Syntax | Hops |
|--------|------|
| `-[:KNOWS]->+` | 1 or more |
| `-[:KNOWS]->*` | **0** or more (includes the start node itself) |
| `-[:KNOWS]->{2,4}` / `{2}` / `{,3}` / `{2,}` | bounded / exact / up to / at least |
| `-[:KNOWS*1..3]->`, `-[:KNOWS*2]->`, `-[*..3]-`, `-[*0..1]->` | Neo4j style |
| `-[:KNOWS*]->` | **1** or more (unlike postfix `*`) |

```cypher
MATCH (a:Person {name: 'Alice'})-[:KNOWS]->+(b) RETURN DISTINCT b.name
MATCH (a:Person {name: 'Alice'})-[:KNOWS*1..3]->(b) RETURN DISTINCT b.name
MATCH (a)-[:KNOWS|WORKS_AT]->{1,2}(b) RETURN a.name, b.name             // type alternatives
MATCH (a)-[e:KNOWS WHERE e.since > 2012]->+(b) RETURN a.name, b.name    // predicate on every hop
MATCH (a)-[:KNOWS*1..3 {since: 2010}]->(b) RETURN b.name                // property map on every hop
MATCH (a {name: 'Alice'})-[e:KNOWS]->+(b) RETURN b.name, size(e)        // e = list of edge IDs
```

- **Trail semantics:** an edge never repeats within one path, but nodes may. Each distinct path is its own row, so use `DISTINCT` or aggregation to collapse them.
- **One quantifier per edge.** You can't combine `[*1..3]` with `{1,3}` on the same edge.
- **The edge variable of a variable-length hop is a list of edge IDs.** `size(e)`, `e[0]`, `e[-1]` and `UNWIND e` work, but `e.since` does not. To read edge properties along the path, name the path and use `relationships(p)`.

### Quantified path patterns

Repeat a one-hop sub-pattern (with its own WHERE) a number of times:

```cypher
MATCH (a:Person {name: 'Alice'}) ((x)-[r:KNOWS]->(y) WHERE r.since > 2012){1,3} (b) RETURN b.name
MATCH (a:Person {name: 'Alice'}) ((x)-[:KNOWS]->(y) WHERE y.age > x.age)+ (b) RETURN b.name   // ages strictly increase
```

Inside the parentheses only a single hop `(node)-[edge]->(node) [WHERE ...]` is allowed. The inner variables (`x`, `r`) become lists.

### Named paths

```cypher
MATCH p = (a:Person {name: 'Alice'})-[:KNOWS]->+(b)
RETURN [n IN nodes(p) | n.name] AS names,
       [r IN relationships(p) | r.since] AS sinces,
       length(p) AS hops
ORDER BY hops
```

- `nodes(p)` and `relationships(p)` return lists of IDs. Read properties through a list comprehension.
- A bare `RETURN p` comes back as a list of `{"type": "node"|"edge", "id": ...}` dicts.
- `count(p)` works; `collect(p)` does not.
- `OPTIONAL MATCH p = ... WHERE ... RETURN p IS NULL` works.
- Shortest paths are covered in `algorithms.md`.

## Joins vs Cartesian Products

```cypher
// Cartesian product (N × M rows) — comma separates independent patterns:
MATCH (p:Person), (c:City) RETURN p.name, c.name

// Join — a shared variable links the patterns:
MATCH (a:Person)-[:KNOWS]->(b), (b)-[:WORKS_AT]->(c) RETURN a.name, b.name, c.name
MATCH (a:Person)-->(i:Interest)<--(b:Person) WHERE a <> b RETURN a.name, b.name

// Join on a value via WHERE (hash join):
MATCH (a:Person), (b:Person) WHERE a.city = b.city AND a <> b RETURN a.name, b.name
```

Cartesian products over large node sets can produce unexpectedly large results — always add WHERE constraints when cross-joining.

## WHERE and operators

```cypher
MATCH (n) WHERE n:Person RETURN n.name                              // label test (also n:A:B, n:A OR n:B, NOT n:A)
MATCH (n:Person) WHERE n.age >= 18 AND n.city = 'London' RETURN n.name
MATCH (n:Person) WHERE n.name STARTS WITH 'A' OR n.name ENDS WITH 'e' OR n.name CONTAINS 'ro' RETURN n.name
MATCH (n:Person) WHERE n.name IN ['Alice', 'Bob'] RETURN n.name
MATCH (n:Person) WHERE 'admin' IN n.tags RETURN n.name              // membership in a list property
MATCH (n:Person) WHERE n.email IS NOT NULL RETURN n.name
MATCH (n:Person) WHERE n.active RETURN n.name                       // bare boolean property
MATCH (n)-[e]->(m) WHERE e:KNOWS AND e.since > 2020 RETURN n.name   // edge type / property
MATCH (p:Person) WHERE (p)-[:LIKES]->() RETURN p.name               // pattern predicate
MATCH (p:Person) WHERE NOT (p)-[:KNOWS]->() RETURN p.name
```

**Operators:**
- Comparison: `= <> != < <= > >=`, plus chained comparisons (`1 < x < 3`).
- Null tests: `IS NULL`, `IS NOT NULL`.
- Boolean: `AND OR XOR NOT`, with three-valued null logic as in Neo4j.
- Membership and strings: `IN`, `STARTS WITH`, `ENDS WITH`, `CONTAINS`.
- Arithmetic: `+ - * / % ^`. `^` returns a Double, and integer `/` truncates.
- Concatenation: `+` and `||` concatenate strings and lists. `'x' + 1` gives `'x1'`.
- Literals: hex literals such as `0x1F` work.

**Not supported:** `=~` regex and `exists(n.prop)`. Use `n.prop IS NOT NULL` instead of the latter.

**Numeric type strictness.** These are deliberate and are rejected at analysis time:
- `Integer = Double` (e.g. `n.age = 30.0`, or `n.score = 2` when `score` is a Double): error "Operands are not valid or compatible types".
- `Double = Double`: error "Equality of types 'Double' and 'Double' is not encouraged…".

Use range comparisons (`n.score > 1.99 AND n.score < 2.01`) or convert with `toInteger`. Mixed-type `<`, `>` and arithmetic are fine.

**Division by zero** raises `EXEC_ERROR`. Int64 overflow wraps silently.

## WITH, DISTINCT, ORDER BY, SKIP/LIMIT, UNION

```cypher
MATCH (n:Person) WITH n WHERE n.age > 28 RETURN n.name
MATCH (n:Person) WITH n.name AS name, n.age AS age ORDER BY age DESC LIMIT 2 RETURN name, age
MATCH (n:Person) WITH n WHERE n.age > 26 WITH n WHERE n.age < 36 RETURN n.name   // chained WITH
MATCH (n:Person) RETURN DISTINCT n.city
MATCH (n:Person) RETURN n.name AS nm ORDER BY size(nm) DESC, nm SKIP 10 LIMIT 10 // expressions, aliases, multiple keys
MATCH (n:Person) RETURN n.name, n.age + 5 AS adjusted, n.price * 1.1
MATCH (n:Person) RETURN *                                                       // all variables (nodes as IDs)
MATCH (p:Person) RETURN p.name AS name UNION MATCH (c:Company) RETURN c.name AS name      // dedup
MATCH (p:Person) RETURN p.name AS name UNION ALL MATCH (c:Company) RETURN c.name AS name  // keep duplicates
```

- **WITH needs aliases.** Non-variable expressions in WITH must be aliased (`WITH n.name AS name`). RETURN doesn't need them; its column is named after the expression text, e.g. `n.name`.
- **SKIP/LIMIT** must be constant expressions.
- **DISTINCT + ORDER BY** can only order by returned columns.
- **Nulls** sort last in ASC and first in DESC.
- **UNION branches** must return the same column names, and each column must keep one value type across branches.
- **EXPLAIN:** `EXPLAIN <query>` returns the compiled plan (IR dump) without running the query.

## Aggregation

Grouping is implicit, as in Neo4j: the non-aggregated return items are the grouping keys.

```cypher
MATCH (a:Person)-[:WORKS_AT]->(c:Company)
RETURN c.name, count(a) AS employees, collect(a.name) AS who, avg(a.age) AS avgAge
ORDER BY employees DESC

MATCH (n:Person) RETURN count(*), count(n.age), sum(n.age), avg(n.age), min(n.age), max(n.age)
MATCH (a)-[w:WORKS_AT]->(c) RETURN w.role, count(DISTINCT c) AS companies
MATCH (n:Person) WITH n.city AS city, count(*) AS c WHERE c > 1 RETURN city, c   // HAVING-style filter
MATCH (n:Person) RETURN n.city, max(n.age) - min(n.age) AS spread                // aggregates inside expressions
MATCH (n:Person) RETURN sum(CASE WHEN n.age > 30 THEN 1 ELSE 0 END) AS seniors
MATCH (n:Person) RETURN collect(n.name)[0..3] AS firstThree
MATCH ()-[r]->() RETURN type(r) AS t, count(*) AS c ORDER BY c DESC              // edge-type histogram
MATCH (n) RETURN labels(n) AS l, count(*) ORDER BY l                             // label histogram
```

**Available aggregates:**
- `count(x)`, `count(*)`, `count(DISTINCT x)`
- `collect(x)`, `collect(DISTINCT x)` (nulls are dropped)
- `sum`
- `avg` (always Double)
- `min`, `max` (numbers, strings, booleans, DateTime, Duration)

There is **no** `stdev`, `percentileCont`, `percentileDisc`, or the other statistical aggregates. Compute those in pandas.

**Rules:**
- Aggregates are invalid in `WHERE`. Filter after a `WITH` instead.
- Aggregates can't be nested (`count(count(*))`).
- A non-aggregated expression next to an aggregate may only read grouping keys.

**Empty input:**
- With no grouping keys, one row is returned: `count` = 0, `sum` = 0, `avg` = null, `collect` = [].
- With grouping keys, no rows are returned.

## Lists, UNWIND, comprehensions, CASE

```cypher
UNWIND [1, 2, 3] AS x RETURN x * 10
UNWIND range(1, 10, 3) AS x RETURN x                              // 1, 4, 7, 10 (inclusive)
MATCH (n:Person) WITH collect(n.name) AS names UNWIND names AS nm RETURN nm
MATCH (n:Person) WITH collect(n) AS ns UNWIND ns AS p RETURN p.name   // unwound nodes stay nodes
MATCH (n:Person) UNWIND n.tags AS t RETURN t, count(*)            // unwind a list property
RETURN [1, 2, 3, 4, 5][0], [1, 2, 3, 4, 5][-1], [1, 2, 3, 4, 5][1..3], [1, 2, 3][10]   // 1, 5, [2,3], null
RETURN [x IN range(1, 6) WHERE x % 2 = 0 | x * x]                 // list comprehension
MATCH (a:Person) RETURN a.name, [(a)-[:KNOWS]->(b) | b.name] AS friends            // pattern comprehension
MATCH (n:Person) WHERE any(t IN n.tags WHERE t = 'vip') RETURN n.name               // all / any / none / single
RETURN size([1, 2]), head([1, 2]), last([1, 2]), tail([1, 2])
```

- **List literals** can contain expressions, mixed types, nested lists and maps.
- **Indexing and slicing** work on literals, list properties, and function results (`labels(n)[0]`).
- **Equality:** lists and maps compare by value.
- **Null handling:** `size(null)`, `head([])` and out-of-range indexes return null.
- **`reduce()` is not supported.** Use `UNWIND` plus an aggregate instead.

```cypher
MATCH (n:Person)
RETURN n.name,
       CASE WHEN n.age >= 35 THEN 'senior' WHEN n.age IS NULL THEN 'unknown' ELSE 'junior' END AS band,
       CASE n.city WHEN 'London' THEN 'UK' WHEN 'Paris' THEN 'FR' ELSE 'other' END AS country,
       CASE n.age WHEN > 30 THEN 'old' WHEN 25, 30 THEN 'mid' WHEN IS NULL THEN '?' END AS ext   // extended CASE tests
```

### Maps

Map literals can be built, returned, compared with `=`, and stored as properties, e.g. `RETURN {name: n.name, tags: n.tags} AS m`.

**Reading from a map is not supported:**
- `m.key`, `m['key']` and `n.mapProp.key` all error.
- Map projection (`n {.name, .age}`) is a parse error.
- `properties(n)` doesn't exist, and `keys()` fails.

Return the individual properties you need instead.

## Subqueries

```cypher
// EXISTS (pattern form or full MATCH form; also usable as a returned value)
MATCH (p:Person) WHERE EXISTS { (p)-[:WORKS_AT]->(:Company {name: 'Acme'}) } RETURN p.name
MATCH (p:Person) WHERE EXISTS { MATCH (p)-[:KNOWS]->(f) WHERE f.age > p.age } RETURN p.name
MATCH (p:Person) WHERE NOT EXISTS { (p)-[:KNOWS]->() } RETURN p.name
MATCH (p:Person) RETURN p.name, EXISTS { (p)-[:LIKES]->() } AS likesSomething

// COUNT
MATCH (p:Person) RETURN p.name, COUNT { (p)-[:KNOWS]->() } AS outDegree ORDER BY outDegree DESC
MATCH (p:Person) WHERE COUNT { MATCH (p)-[:KNOWS]-(x) RETURN x } >= 2 RETURN p.name

// CALL subqueries: per-row top-k, per-row aggregates, UNION inside
MATCH (p:Person) CALL (p) { MATCH (p)-[:KNOWS]->(f) RETURN f.name AS friend ORDER BY f.name LIMIT 1 } RETURN p.name, friend
MATCH (p:Person) CALL { WITH p MATCH (p)-[:KNOWS]->(f) RETURN count(f) AS deg } RETURN p.name, deg
MATCH (p:Person) OPTIONAL CALL (p) { MATCH (p)-[:WORKS_AT]->(c) RETURN c.name AS co } RETURN p.name, co
CALL { MATCH (p:Person) RETURN p.name AS nm UNION MATCH (c:Company) RETURN c.name AS nm } RETURN nm ORDER BY nm
```

- **Dropped rows:** a non-optional `CALL` drops outer rows for which the body returns nothing.
- **Imported variables:** a CALL body can't return an imported variable under its own name. Alias it (`RETURN p AS me`).
- **Not supported:**
  - `COLLECT { ... }`. Use `collect()` in a CALL body or a pattern comprehension instead.
  - `CALL (*) { }`.
  - `CALL { } IN TRANSACTIONS`.
  - `OPTIONAL CALL` on a *procedure*.

## Built-in functions

This is the complete list. **Function names are case-sensitive** (`toInteger`, not `TOINTEGER`), except `count` and `collect`.

| Category | Functions | Notes |
|----------|-----------|-------|
| Entity | `id(x)`, `labels(n)`, `type(r)`, `startNode(r)`, `endNode(r)` | `labels` returns a **list**. `type` returns a string (`edgeType()` was removed). `startNode(r).name` works |
| Path | `nodes(p)`, `relationships(p)`, `length(p)` | Lists of IDs; read properties via a comprehension |
| List | `size`, `length`, `head`, `last`, `tail`, `range(start, end[, step])` | `size`/`length` also work on strings. `range` bounds are inclusive |
| Predicates | `all`, `any`, `none`, `single` `(x IN list WHERE …)` | |
| Conversion | `toInteger`, `toFloat`, `toString`, `toBoolean`, `coalesce(a, b, …)` | `toInteger`/`toFloat` accept strings, ints and doubles; a bad string gives null. `toBoolean` accepts strings only. `coalesce` arguments must share one type |
| Temporal | `datetime(isoString)`, `datetime(epochSeconds)`, `datetime()`, `duration({days: 2, hours: 3})`, `duration(micros)` | See Data Types |
| Embedding | `cosine_similarity(a, b)`, `euclidean_distance(a, b)` | Arguments must be Embeddings, not lists (see `algorithms.md`) |
| Aggregate | `count`, `collect`, `sum`, `avg`, `min`, `max` | |

**Not available** (each errors with "Function 'X' does not exist"):
- **String:** `toUpper`, `toLower`, `substring`, `split`, `replace`, `trim`, `left`, `right`, `reverse`.
- **Math:** `abs`, `round`, `floor`, `ceil`, `sqrt`, `log`, `exp`, `rand`, `sign`, `pi`.
- **Temporal:** `date()`, `timestamp()`.
- **Other:** `properties`, `elementId`, `exists()`, `reduce`.

Post-process in pandas instead.

## LOAD CSV

`LOAD CSV` streams rows from a CSV file under the server's `data/` directory (`<turing-dir>/data/`). Absolute paths are re-rooted inside it.

```cypher
LOAD CSV 'people.csv' WITH HEADERS AS row RETURN row.name, toInteger(row.age) AS age     // by column name
LOAD CSV 'people.csv' AS row RETURN row[0], row[1]                                       // by 0-based index
LOAD CSV 'people.csv' WITH HEADERS AS row MATCH (p:Person {name: row.name}) RETURN p.name, row.city   // join with the graph
LOAD CSV 'people.csv' WITH HEADERS AS row WITH row.name AS nm, toInteger(row.age) AS a WHERE a > 30 RETURN nm, a
LOAD CSV 'bad.csv' WITH HEADERS ON ERROR SKIP AS row RETURN row.a                        // skip malformed rows (default ON ERROR FAIL)
```

**Syntax limits:**
- The syntax is `LOAD CSV '<file>' [WITH HEADERS] [ON ERROR SKIP|FAIL] AS row`. The Neo4j forms `LOAD CSV WITH HEADERS FROM 'file:///…'` and `FIELDTERMINATOR` are not supported; files must be comma-separated.
- Every field is a **string**. Convert with `toInteger` / `toFloat` / `datetime`.
- `row` can't be returned whole. Read fields as `row.col` or `row[<literal index>]`.
- Without `WITH HEADERS`, the header line comes back as a data row.

**Composition:**
- MATCH, WITH/WHERE, aggregates, ORDER BY and UNWIND all compose with LOAD CSV.
- **OPTIONAL MATCH bug:** an OPTIONAL MATCH directly after LOAD CSV that uses `row.x` inside the pattern hits an internal error. Project the fields first: `... AS row WITH row.name AS nm OPTIONAL MATCH (p:Person {name: nm}) ...`.

To **build a graph** from CSV, see `writing.md`.

## Data types and how results come back

| Type | Cypher literal | HTTP `column_types` | pandas dtype |
|------|----------------|---------------------|--------------|
| String | `'Alice'` / `"Alice"` | `String` | `string` |
| Integer | `30`, `0x1F` | `Int64` | `Int64` |
| Unsigned integer | — (IDs, `count`, `length(p)`) | `UInt64` | `UInt64` |
| Double | `3.14`, `1e3` | `Double` | `float64` |
| Boolean | `true` | `Bool` | `boolean` |
| List | `[1, 'a', [2]]` | `List` / `ListElement` | object (Python list) |
| Map | `{a: 1}` | `Map` | object (dict) |
| Embedding | `(1.2, 2.0, 0.0)` | `Embedding` | object (list over HTTP, numpy float32 array over native/embedded) |
| DateTime | `datetime('2024-03-15T10:30:00Z')` | `DateTime` | `datetime64[us, UTC]` |
| Duration | `duration({days: 2})` | `Duration` | `timedelta64[us]` |
| Node / edge | `RETURN n` | `UInt64` | `UInt64` (internal ID) |
| Named path | `RETURN p` | `EntityList` | object (list of `{type, id}` dicts) |

**Literals:**
- Strings use single or double quotes, with backslash escapes (`\'`, `\"`).
- **Backticks quote identifiers**, not strings: `` `my var` ``.
- **Embedding vs list:** an embedding literal uses parentheses and needs at least 2 numeric items, e.g. `(1.0, 2.0)`. Square brackets `[...]` make a List, which is a different type.

**Temporal values:**
- `datetime` values are normalized to UTC; a date-only string means midnight.
- Arithmetic works: datetime ± duration, datetime − datetime, duration × number.
- Components work: `d.year`, `d.month`, `d.day`, `d.hour`, …, `dur.days`, `dur.hours`, …
- `min`/`max` and `<`/`>` work on temporal values.

**Precision over HTTP:** the JSON encoder prints Doubles with **6 decimal places** (`3.14159265` → `3.141593`, `1e-9` → `0.000000`). If you need full precision, scale the value in the query or use the embedded backend.

## Gotchas

- `RETURN n` gives an integer ID, not a node object or property map. Project the properties you need (`n.name, n.age`).
- No parameters (`$x`). Inline literals and escape quotes.
- Aggregates are invalid in WHERE. Use `WITH … WHERE`.
- Comma-separated patterns are cartesian products, not joins. Make sure you intended that rather than a join on a shared variable.
- Postfix `->*` means 0+ hops, but bracket `-[*]->` means 1+ hops.
- `Integer = Double` and `Double = Double` comparisons are rejected. Use ranges.
- Function names are case-sensitive, and unknown property names silently read as null.
- Keywords are case-insensitive (`MATCH` == `match`). Labels, edge types and property names are case-sensitive.
- Only `//` starts a comment; `-- text` is parsed as part of the query. `s3` is a reserved word (S3 commands), so don't use it as a variable name.
