---
name: turingdb-querying
description: Use when reading data from TuringDB v3 — MATCH/OPTIONAL MATCH patterns, label expressions, variable-length and quantified paths, WHERE, WITH, NEXT, WHEN conditional queries, aggregation, UNWIND, lists, maps, CASE, reduce, EXISTS/COUNT/COLLECT/CALL subqueries, UNION, built-in functions, label/edge-type parameters, LOAD CSV, and result types. Does not cover writes (see writing.md) or shortest path / vector search (see algorithms.md).
---

# TuringDB: Querying

Read queries run directly against the current graph state — no change workflow needed:

```python
df = client.query("MATCH (n:Person) RETURN n.name, n.age")
```

**Before writing queries, know these limits:**
- **Parameters only name labels and edge types** (see Parameters below). A `$name` used as a value fails with "Not implemented: Parameters", so build the query text with inlined literals and escape quotes yourself.
- **One statement per request.** `;` chaining is rejected with "Multiple queries not yet supported" (a single trailing `;` is fine). To run steps in order within one request, use `NEXT` (below).
- **Comments:** `//` line comments work. SQL-style `--` comments and `/* */` comments do not.

## MATCH

```cypher
MATCH (n) RETURN count(n)
MATCH (n:Person) RETURN n.name, n.age
MATCH (n:Person {name: 'Alice', age: 30}) RETURN n.name            // inline property map
MATCH (n:Person:Employee) RETURN n.name                           // has BOTH labels (same as :Person&Employee)
MATCH (a:Person)-[:KNOWS]->(b:Person) RETURN a.name, b.name
MATCH (a)-[e:KNOWS {since: 2010}]->(b) RETURN a.name, e.since, b.name
MATCH (a)<-[e]-(b) RETURN a.name, b.name                          // reverse direction
MATCH (a)-[e]-(b) RETURN a.name, b.name                           // undirected (also --, <-[e]->, <-->)
MATCH (a)-[r:KNOWS|WORKS_AT]->(b) RETURN a.name, type(r), b.name  // edge type alternatives (also :KNOWS|:WORKS_AT)
MATCH (a:Person WHERE a.age > 30)-[r:KNOWS WHERE r.since > 2015]->(b) RETURN b.name  // inline WHERE (variable must be named)
MATCH (a {name: 'Alice'}) MATCH (a)-[:KNOWS]->(b) RETURN b.name   // multiple MATCH clauses
MATCH (n:Person) ORDER BY n.age LIMIT 2 RETURN n.name             // TuringDB extension: ORDER BY/SKIP/LIMIT on MATCH
```

**Label expressions** work on nodes and edges, in patterns and in WHERE:

```cypher
MATCH (n:Person|Company) RETURN n.name                  // either label
MATCH (n:!Person) RETURN n.name                         // any label but Person
MATCH (n:Person&Employee) RETURN n.name                 // both labels
MATCH (n:(Person&!Employee)|Company) RETURN n.name      // nested, with parentheses
MATCH (n) WHERE n:Person|Company RETURN n.name          // in WHERE
MATCH (a)-[r:!KNOWS]->(b) RETURN type(r), count(*)      // any edge type but KNOWS
```

- **No wildcard:** the label wildcard `:%` is a parse error.
- **Write patterns:** in CREATE and MERGE, labels can only be joined by `:` or `&`.
- **Unknown labels and properties don't error.** A label that doesn't exist matches nothing, and a misspelled property silently reads as null.

### Parameters (labels and edge types only)

`client.query(query, query_parameters)` takes a dict of parameters. The server accepts a `$name` (or a numbered `$0`) **only where a label or an edge type goes**: in MATCH, CREATE and MERGE patterns and in label expressions.

```python
client.query("MATCH (n:$lbl) RETURN n.name", {"lbl": "Person"})
client.query("MATCH (n:$lbls) RETURN n.name", {"lbls": ["Person", "Employee"]})   # a list means every label
client.query("MATCH (a)-[r:$t]->(b) RETURN a.name, b.name", {"t": "WORKS_AT"})
client.query("MATCH (n:Person|$other) RETURN n.name", {"other": "Company"})
```

- **No value parameters:** `WHERE n.name = $name`, `{name: $name}`, `SET n.x = $v` and `RETURN $x` all fail with "Not implemented: Parameters". Inline values as literals.
- **Undefined parameters** fail at run time with `Parameter $lbl is not defined`.
- **Values:** a label or edge-type parameter holds a string, or a list of strings to require several labels.

### OPTIONAL MATCH

```cypher
MATCH (a:Person) OPTIONAL MATCH (a)-[:WORKS_AT]->(c:Company) RETURN a.name, c.name   // c.name null if no match
MATCH (a:Person) OPTIONAL MATCH (a)-[:KNOWS]->(b) RETURN a.name, count(b)            // 0 for no match
```

The WHERE of an OPTIONAL MATCH belongs to the optional part, so rows are kept, as in Neo4j. An unmatched node or edge returned bare (`RETURN c`) is null.

### Addressing nodes by native node ID (the fast path)

Every node has a native node ID: the integer you get from `RETURN n` or `id(n)`. **Whenever you already know which node(s) you want, address them by native ID: `WHERE n = <id>` (or `WHERE id(n) = <id>`).**
- **How it runs:** a literal ID becomes a constant scan of exactly that node (`const_scan_nodes`), and IDs read from a row are fetched directly (`fetch_nodes`).
- **Don't re-look them up** through a user-level key property such as `n.pid`, `n.uuid` or `n.doc_id`.

```cypher
MATCH (n) WHERE n = 4 RETURN n.name                                   // direct seek (id(n) = 4 is the same)
MATCH (n)-[:KNOWS]->(m) WHERE n = 4 RETURN m.name                     // seek, then expand
UNWIND [4, 17, 42] AS x MATCH (n) WHERE n = x RETURN x, n.name        // batch: one direct fetch per ID
UNWIND [4, 17, 42] AS x MATCH (n) WHERE n = x SET n.flag = true       // batch write by ID (inside a change)
UNWIND [[4, 17], [42, 8]] AS p MATCH (a) WHERE a = p[0] MATCH (b) WHERE b = p[1] RETURN a.name, b.name   // two nodes per row
```

Measured on a 200k-node, 400k-edge graph (server-side time):

| Lookup | Time |
|---|---|
| `WHERE n = <id>` or `WHERE id(n) = <id>` | **0.14 ms** (constant scan) |
| `UNWIND <1,000 IDs> AS x` + `WHERE n = x` | **≈3 ms total (≈3 µs per node)**, whatever the list source |
| `UNWIND <10 ID pairs> AS p` + `a = p[0]`, `b = p[1]` | 0.3 ms |
| `WHERE n IN [<10 IDs>]` | 3.8 ms (full node scan; grows with graph size) |
| `UNWIND <1,000 keys> AS x` + `MATCH (n:P {pid: x})` | ≈290 ms (≈0.3 ms **per node**, ≈100× slower) |

**Rules:**
- **Batch with `UNWIND <IDs> AS x MATCH (n) WHERE n = x`, not `IN`.** `WHERE n IN [..]` and `WHERE id(n) IN [..]` scan every node.
- **Any list source works.** A literal list, a `WITH` variable, `collect(id(n))`, `range(...)`, duplicates, returning `x`, a label on `n`, and an expansion after the match all keep the direct fetch.
  - **IDs computed per row are fine too:** `a = p[0] AND b = p[1]`, or two MATCHes on two values of one row.
  - **Check with `EXPLAIN (db)`:** the plan should show `const_scan_nodes` or `fetch_nodes`, not `scan_nodes` followed by a filter, and no `cross_product`.
- **Get the IDs once, reuse them.** Fetch them with whatever filter you need (`MATCH (n:Person {email: 'a@x.org'}) RETURN id(n)`, or `RETURN n`), keep them in Python, and address those nodes by ID from then on.
  - **Key lookups:** a single key lookup with a constant (`{pid: 7}`) is fine. It's per-row key lookups (`UNWIND keys … {pid: x}`, LOAD CSV joins) that are slow.
  - **Never match two nodes per row by key property** (`MATCH (a:P {pid: p[0]}) MATCH (b:P {pid: p[1]})`). Each row scans the whole label once per endpoint, and a property index doesn't help. That is about 7.5 s for 1,000 pairs on 200k nodes, against 3.8 ms by native ID. Map the keys to native IDs first.
- **Vector search results:** `VECTOR SEARCH … YIELD ids` binds `ids` directly as nodes. Use them as is (`ids.title`, `MATCH (ids)-[...]->(...)`). Re-matching with `MATCH (n:Label) WHERE n = ids` is also a direct fetch now; use it only to keep the hits that carry a label.

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
MATCH (a {name: 'Alice'})-[e:KNOWS]->+(b) RETURN b.name, size(e)        // e = list of edges
MATCH (a {name: 'Alice'})-[e:KNOWS]->+(b) RETURN b.name, [r IN e | r.since] AS sinces   // edge properties along the path
```

- **Trail semantics:** an edge never repeats within one path, but nodes may. Each distinct path is its own row, so use `DISTINCT` or aggregation to collapse them.
- **One quantifier per edge.** You can't combine `[*1..3]` with `{1,3}` on the same edge.
- **The edge variable of a variable-length hop is a list of edges.** `size(e)`, `e[0]`, `e[-1]`, `UNWIND e` and comprehensions (`[r IN e | r.since]`) work. `e.since` fails: "Variable 'e' binds the list of a variable-length path, not a single entity".

### Quantified path patterns

Repeat a sub-pattern (with its own WHERE) a number of times. The sub-pattern can span several hops:

```cypher
MATCH (a:Person {name: 'Alice'}) ((x)-[r:KNOWS]->(y) WHERE r.since > 2012){1,3} (b) RETURN b.name
MATCH (a:Person {name: 'Alice'}) ((x)-[:KNOWS]->(y) WHERE y.age > x.age)+ (b) RETURN b.name   // ages strictly increase
MATCH (a:Person {name: 'Alice'}) ((x)-[:KNOWS]->(y)-[:KNOWS]->(z)){1,2} (b) RETURN b.name    // two-hop body, 2 or 4 hops
MATCH (a:Person {name: 'Alice'}) ((x)-[r:KNOWS]->(y)){1,3} (b)
RETURN b.name, [n IN y | n.name] AS via, size(r) AS hops                                    // group variables are lists
```

The inner variables (`x`, `y`, `r`) are group variables: each becomes a list, one element per repetition. Use them in comprehensions, `size()`, `UNWIND`, sort keys and list literals.

### Named paths

```cypher
MATCH p = (a:Person {name: 'Alice'})-[:KNOWS]->+(b)
RETURN [n IN nodes(p) | n.name] AS names,
       [r IN relationships(p) | r.since] AS sinces,
       length(p) AS hops
ORDER BY hops
```

- `nodes(p)` and `relationships(p)` return lists of nodes and edges. Read properties through a list comprehension, or fold them with `reduce(t = 0, r IN relationships(p) | t + r.distance)`.
- A bare `RETURN p` comes back as a list of `{"type": "node"|"edge", "id": ...}` dicts.
- `count(p)` and `RETURN DISTINCT p` (grouping on a path) work; `collect(p)` does not.
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
MATCH (n) WHERE n:Person RETURN n.name                              // label test (also n:A|B, n:A&B, n:!A, NOT n:A)
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
- Membership and strings: `IN`, `STARTS WITH`, `ENDS WITH`, `CONTAINS`, `IS [NFC|NFD|NFKC|NFKD] NORMALIZED`.
- Arithmetic: `+ - * / % ^`. `^` returns a Double, and integer `/` truncates.
- Concatenation: `+` and `||` concatenate strings and lists, and `list + x` appends an element. `'x' + 1` gives `'x1'`, and `'x' + true` gives `'xtrue'`.
- Literals: hex literals such as `0x1F` work. Keywords can be used as property names and map keys (`m.match`, `{return: 1}`).

**Not supported:** `=~` regex and `exists(n.prop)`. Use `n.prop IS NOT NULL` instead of the latter, and string functions (`toLower(n.name) STARTS WITH 'a'`) instead of regex.

**Numeric equality:**
- `Integer = Double` compares the exact values: `n.age = 30.0` matches an Integer 30, and `n.age IN [30.0, 25]` works.
- `Double = Double` is still rejected at analysis time ("Equality of types 'Double' and 'Double' is not encouraged…"). Use range comparisons (`n.score > 1.99 AND n.score < 2.01`).
- **Cross-type ordering:** comparing values with no common order (`1 < 'a'`) gives null.

**Division by zero** raises `EXEC_ERROR`, for Doubles too (`0.0 / 0.0`). Int64 overflow wraps silently.

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
- **EXPLAIN:** `EXPLAIN <query>` returns the compiled plan (IR dump) without running the query. `EXPLAIN (db) <query>` limits it to the `db` stage, which is the one to read for scans and joins. Other stages are `ast`, `vdg`, `codegen`, `passes`, `nl` and `all`.

### NEXT: chain queries in one request

`NEXT` runs the query before it, then feeds its RETURN columns to the query after it as the incoming rows. Use it instead of `;` to run steps in order:

```cypher
MATCH (n:Person) RETURN n.name AS nm, n.age AS age
NEXT
RETURN nm ORDER BY nm LIMIT 2

MATCH (n:Person) RETURN n.city AS city, count(*) AS c
NEXT
RETURN max(c) AS biggestCity
```

- **Aliases are required:** every expression in a RETURN followed by `NEXT` must be aliased with `AS`.
- **Writes:** a later segment sees what an earlier segment wrote (`CREATE (x:N) RETURN x NEXT MATCH (y:N) RETURN count(y)`).

### WHEN: conditional queries

`WHEN <condition> THEN <query> [WHEN … THEN …] [ELSE <query>]` runs the first branch whose condition is true. Branches can read or write, and must return the same columns.

```cypher
WHEN true THEN RETURN 1 AS x ELSE RETURN 2 AS x

// After NEXT: the condition reads the incoming row
MATCH (n:Person) RETURN n.name AS nm, n.age AS age
NEXT
WHEN age > 30 THEN RETURN nm, 'senior' AS band
ELSE RETURN nm, 'junior' AS band

// Per row, inside a CALL subquery
MATCH (n:Person)
CALL (n) { WHEN n.age > 30 THEN RETURN 'old' AS b ELSE RETURN 'young' AS b }
RETURN n.name, b

{ WHEN true THEN RETURN 1 AS x } UNION { WHEN true THEN RETURN 2 AS x }   // combine with UNION
```

**Where WHEN can go:** a WHEN can start a query, follow `NEXT`, or form a `CALL { }` body. It can't follow `MATCH … WITH` directly; put `NEXT` in between. A branch can hold a NEXT chain in braces: `WHEN c THEN { MATCH … RETURN … NEXT RETURN … }`.

**Not supported:** UNION inside a WHEN branch, and WHEN inside EXISTS, COUNT or COLLECT bodies.

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
- `min`, `max`: numbers, strings, booleans, DateTime, Duration, lists, and mixed-type values (ordered as in Neo4j)
- `stDev` (sample), `stDevP` (population)
- `percentileCont(x, p)`, `percentileDisc(x, p)`; `p` may also be read off the row

```cypher
MATCH (n:Person) RETURN stDev(n.age) AS sd, percentileCont(n.age, 0.5) AS median, percentileDisc(n.age, 0.9) AS p90
```

**Rules:**
- Aggregates are invalid in `WHERE`. Filter after a `WITH` instead.
- Aggregates can't be nested (`count(count(*))`).
- A non-aggregated expression next to an aggregate may only read grouping keys.

**Empty input:**
- With no grouping keys, one row is returned: `count` = 0, `sum` = 0, `avg` = null, `collect` = [].
- With grouping keys, no rows are returned.

## Lists, UNWIND, comprehensions, reduce, CASE

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
RETURN size([1, 2]), head([1, 2]), last([1, 2]), tail([1, 2]), reverse([1, 2])
RETURN reduce(acc = 0, x IN [1, 2, 3] | acc + x) AS total                          // fold a list
```

- **List literals** can contain expressions, mixed types, nested lists and maps.
- **Indexing and slicing** work on literals, list properties, and function results (`labels(n)[0]`).
- **Equality:** lists and maps compare by value.
- **Null handling:** `size(null)`, `head([])` and out-of-range indexes return null.
- **Mixed-type lists:** elements read from a mixed-type list (`row[1]` of `['Brest', 139000]`) have type Any. Arithmetic, string functions and conversions work on them. To write one as a property, convert it first (`toInteger(row[1])`).

```cypher
MATCH (n:Person)
RETURN n.name,
       CASE WHEN n.age >= 35 THEN 'senior' WHEN n.age IS NULL THEN 'unknown' ELSE 'junior' END AS band,
       CASE n.city WHEN 'London' THEN 'UK' WHEN 'Paris' THEN 'FR' ELSE 'other' END AS country,
       CASE n.age WHEN > 30 THEN 'old' WHEN 25, 30 THEN 'mid' WHEN IS NULL THEN '?' END AS ext   // extended CASE tests
```

CASE branches can return lists, maps, DateTimes and Durations, and branches of different types can be mixed (`CASE WHEN c THEN 1 ELSE 'x' END`). The same holds for `coalesce(n.score, 'none')`.

### Maps

Map literals can be built, returned, compared, sorted, tested for null, and stored as properties. Reading them works too:

```cypher
WITH {a: 1, b: 'x'} AS m RETURN m.a, m.b                          // static key
WITH {a: 1, b: 2} AS m UNWIND keys(m) AS k RETURN k, m[k]         // dynamic key from a variable
WITH [{a: 1}, {a: 2}] AS ms RETURN ms[1].a, [m IN ms | m.a]       // map in a list
WITH {xs: [1, 2, 3]} AS m RETURN m.xs[0], size(m.xs)              // list in a map
UNWIND [{name: 'Brest', pop: 139000}] AS row RETURN row.name, row.pop
MATCH (n:Person) RETURN properties(n) AS props, keys(n) AS ks     // all properties of a node or edge
MATCH (n:Person) WHERE 'email' IN keys(n) RETURN n.name           // property presence test
MATCH (n:Person) RETURN {name: n.name, age: n.age} AS m           // instead of map projection
```

- **Literal keys:** `m['a']` with a literal key is rejected ("A map key known in the query is written m.key"). Write `m.a`; brackets are for keys held in a variable.
- **No map projection:** `n {.name, .age}` and `n {.*}` are parse errors. Build the map literal instead, or use `properties(n)`.
- **Type-erased values:** map values have type Any. They work in expressions, but writing one as a property needs a conversion (see `writing.md`).

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

// COLLECT: the body's single column as a list
MATCH (p:Person) RETURN p.name, COLLECT { MATCH (p)-[:KNOWS]->(f) RETURN f.name } AS friends
MATCH (p:Person) WHERE size(COLLECT { MATCH (p)-[:KNOWS]-(f) RETURN f.name }) >= 2 RETURN p.name

// CALL subqueries: per-row top-k, per-row aggregates, UNION inside
MATCH (p:Person) CALL (p) { MATCH (p)-[:KNOWS]->(f) RETURN f.name AS friend ORDER BY f.name LIMIT 1 } RETURN p.name, friend
MATCH (p:Person) CALL { WITH p MATCH (p)-[:KNOWS]->(f) RETURN count(f) AS deg } RETURN p.name, deg
MATCH (p:Person) CALL (*) { MATCH (p)-[:KNOWS]->(f) RETURN count(f) AS deg } RETURN p.name, deg   // import everything
MATCH (p:Person) OPTIONAL CALL (p) { MATCH (p)-[:WORKS_AT]->(c) RETURN c.name AS co } RETURN p.name, co
CALL { MATCH (p:Person) RETURN p.name AS nm UNION MATCH (c:Company) RETURN c.name AS nm } RETURN nm ORDER BY nm
```

- **Dropped rows:** a non-optional `CALL` drops outer rows for which the body returns nothing.
- **Imported variables:** a CALL body can't return an imported variable under its own name. Alias it (`RETURN p AS me`).
- **Conditional bodies:** a CALL body can be a `WHEN … THEN … ELSE …` (see WHEN above).
- **Not supported:**
  - `CALL { } IN TRANSACTIONS`.
  - `OPTIONAL CALL` on a *procedure*.
  - WHEN inside an EXISTS, COUNT or COLLECT body.

## Built-in functions

This is the complete list. **Function names are case-insensitive** (`toInteger`, `TOINTEGER` and `ToInteger` all work).

| Category | Functions | Notes |
|----------|-----------|-------|
| Entity | `id(x)`, `labels(n)`, `type(r)`, `startNode(r)`, `endNode(r)`, `keys(x)`, `properties(x)` | `labels` and `keys` return **lists**. `properties` returns a map of a node, an edge or a map. `type` returns a string. `startNode(r).name` works |
| Path | `nodes(p)`, `relationships(p)`, `length(p)` | Lists of nodes / edges; read properties via a comprehension |
| List | `size`, `length`, `head`, `last`, `tail`, `reverse`, `range(start, end[, step])` | `size`/`length` also work on strings. `range` bounds are inclusive |
| Fold | `reduce(acc = init, x IN list \| expr)` | |
| Predicates | `all`, `any`, `none`, `single` `(x IN list WHERE …)` | Stop at the first element that decides the answer |
| String | `toUpper`, `toLower`, `trim`, `ltrim`, `rtrim`, `substring(s, start[, len])`, `left(s, n)`, `right(s, n)`, `split(s, sep)`, `replace(s, from, to)`, `reverse`, `char_length` / `character_length`, `normalize(s[, NFC\|NFD\|NFKC\|NFKD])` | 0-based positions; a negative `start` or `len` raises EXEC_ERROR. `normalize` gives null for a non-string |
| Math | `abs`, `sign`, `round(x)`, `floor`, `ceil`, `sqrt`, `exp`, `log`, `log10`, `e()`, `pi()`, `sin`, `cos`, `tan`, `cot`, `asin`, `acos`, `atan`, `haversin`, `radians`, `degrees` | `round`/`floor`/`ceil` return Doubles. `log(-1)` and `sqrt(-1)` give NaN |
| Conversion | `toInteger`, `toFloat`, `toString`, `toBoolean`, `toIntegerOrNull`, `toFloatOrNull`, `toStringOrNull`, `toBooleanOrNull`, `coalesce(a, b, …)` | Strict forms accept strings, numbers and booleans (`toBoolean(1)`, `toInteger(true)`), and give null for a bad string. The `OrNull` forms give null for any unconvertible value. `toString` formats DateTimes (`2024-03-15T10:30:00Z`) and Durations (`PT48H`) |
| Temporal | `datetime(isoString)`, `datetime(epochSeconds)`, `datetime()`, `duration({days: 2, hours: 3})`, `duration(micros)` | See Data Types |
| Embedding | `cosine_similarity(a, b)`, `euclidean_distance(a, b)` | Arguments must be Embeddings, not lists (see `algorithms.md`) |
| Aggregate | `count`, `collect`, `sum`, `avg`, `min`, `max`, `stDev`, `stDevP`, `percentileCont`, `percentileDisc` | |

**Not available** (each errors with "Function 'X' does not exist"):
- **String:** `btrim`, `lpad`/`rpad`, and `=~` regex.
- **Math:** `rand`, `atan2`, `isNaN`, and the two-argument `round(x, digits)` ("Invalid arguments").
- **Temporal:** `date()`, `localdatetime()`, `timestamp()`.
- **Other:** `elementId`, `exists()`.

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

**Composition:** MATCH, OPTIONAL MATCH, WITH/WHERE, aggregates, ORDER BY, UNWIND and subqueries all compose with LOAD CSV and can read `row.x`.

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

**Doubles over HTTP** keep full precision (`0.1 + 0.2` → `0.30000000000000004`). Non-finite values are written as `NaN`, `Infinity` and `-Infinity`, and the SDK reads them back as float `nan` / `inf`. NaN sorts after every number.

## Gotchas

- `RETURN n` gives an integer ID, not a node object or property map. Project the properties you need (`n.name, n.age`) or use `properties(n)`.
- `$parameters` only name labels and edge types. Inline values as literals and escape quotes.
- Aggregates are invalid in WHERE. Use `WITH … WHERE`.
- Comma-separated patterns are cartesian products, not joins. Make sure you intended that rather than a join on a shared variable.
- Postfix `->*` means 0+ hops, but bracket `-[*]->` means 1+ hops.
- `Double = Double` comparisons are rejected. Use ranges. `Integer = Double` is fine.
- Unknown property names silently read as null.
- Keywords and function names are case-insensitive (`MATCH` == `match`, `AVG` == `avg`). Labels, edge types and property names are case-sensitive.
- Only `//` starts a comment; `-- text` is parsed as part of the query. `s3` is a reserved word (S3 commands), so don't use it as a variable name.
