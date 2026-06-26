---
name: turingdb-querying
description: Use when reading data from TuringDB — MATCH patterns, WHERE filtering, joins, ordering, and result shaping. Does not cover writes (see writing.md) or algorithms (see algorithms.md).
---

# TuringDB: Querying

Read queries run directly against the current graph state — no change workflow needed:

```python
df = client.query("MATCH (n:Person) RETURN n.name, n.age")
```

## MATCH

```cypher
MATCH (n) RETURN n
MATCH (n:Person) RETURN n.name, n.age
MATCH (n:Person {name: 'Alice'}) RETURN n.age          -- inline property filter
MATCH (a:Person)-[:KNOWS]->(b:Person) RETURN a.name, b.name
MATCH (a)-[e]->(b) RETURN a, e, b                      -- capture edge variable
MATCH (a:Person)-[e:KNOWS]->(b:Person) RETURN a, e, b
MATCH (a)<-[e]-(b) RETURN a, b                         -- reverse direction
MATCH (a)-[e]-(b) RETURN a, b                          -- undirected
MATCH (a)-[e1]->(b)-[e2]->(c) RETURN a, c              -- multi-hop
```

**Variable-length paths** use a postfix quantifier on the segment (`->+`, `->*`, `->{n,m}`) — *not* the Neo4j inline `-[*1..3]-` form:

```cypher
MATCH (a:Person)-[e]->+(b:Person) RETURN a, b          -- one or more hops
MATCH (a:Person)-[e]->*(b:Person) RETURN a, b          -- zero or more hops
MATCH (a:Person)-[e]->{2,4}(b:Interest) RETURN a, b    -- bounded: 2 to 4 hops
```

Quantifiers: `+` (1+), `*` (0+), `{n,m}` (bounded).

## WHERE

```cypher
MATCH (n) WHERE n:Person RETURN n                           -- filter on label
MATCH (n) WHERE n.name = 'Alice' RETURN n
MATCH (n) WHERE n.age >= 18 AND n.city = 'London' RETURN n.name
MATCH (n) WHERE n.med = 'Aspirin' OR n.med = 'Ibuprofen' RETURN n.name
MATCH (n) WHERE n.institution IS NOT NULL RETURN n
MATCH (n) WHERE n = 0 OR n = 1 RETURN n                     -- match by internal node ID
MATCH (n)-[e]->(m) WHERE e:KNOWS RETURN n.name, m.name      -- filter on edge label
MATCH (n)-[e]->(m) WHERE e.since > 2020 RETURN n.name       -- filter on edge property
```

Operators: `=`, `<>`, `<`, `<=`, `>`, `>=`, `IS NULL`, `IS NOT NULL`. Boolean combiners: `AND`, `OR`. Arithmetic in expressions: `+ - * /`.

Inline property filter `{name: 'Alice'}` is exactly equivalent to `WHERE n.name = 'Alice'`.

## Joins vs Cartesian Products

```cypher
-- Cartesian product (N × M rows) — comma separates independent patterns:
MATCH (p:Person), (c:City) RETURN p.name, c.name

-- Join — shared variable links the patterns (TuringDB runs a hash join):
MATCH (a:Person)-->(i:Interest)<--(b:Person)
WHERE a.name <> b.name
RETURN a.name, b.name

-- Join on a value via WHERE (also a hash join):
MATCH (a:Person), (b:Person) WHERE a.hasPhD = b.hasPhD RETURN a.name, b.name

-- Mixed: join on one pattern, cartesian on another:
MATCH (a:Person)-->(i:Interest), (c:Category)
WHERE c.name = 'Cat1'
RETURN a.name, i.name, c.name
```

Cartesian products over large node sets can produce unexpectedly large results — always add WHERE constraints when cross-joining.

## ORDER BY / SKIP / LIMIT

```cypher
MATCH (n:Person) RETURN n.name, n.age ORDER BY n.age
MATCH (n:Person) RETURN n.name, n.age ORDER BY n.age DESC
MATCH (n:Person) RETURN n.name, n.age, n.city ORDER BY n.city, n.age DESC
MATCH (n:Person) RETURN n.name, n.age ORDER BY n.age DESC SKIP 10 LIMIT 10
```

## Expression Evaluation in RETURN

```cypher
MATCH (n) RETURN n.price * 1.1
MATCH ()-[r]->() RETURN r.a / r.b
MATCH (n) RETURN n.val + n.tax
MATCH (n) RETURN n.age + 5 AS adjusted_age                 -- alias with AS
```

## Built-in Functions

**Scalar functions** (usable in both RETURN and WHERE):

| Function | Example | Notes |
|----------|---------|-------|
| `labels(n)` | `RETURN labels(n), n.name` | Node label as string |
| `edgeType(e)` | `RETURN edgeType(e)` | Edge type as string |
| `toInteger(expr)` | `WHERE n.year > toInteger("2020")` | Parse string to int |
| `toFloat(expr)` | `RETURN n.price * toFloat("1.07")` | Parse string to float |
| `toBoolean(expr)` | `RETURN toBoolean("true")` | Parse string to bool |
| `cosine_similarity(a, b)` | `RETURN cosine_similarity(n.emb, (0.4, 0.3))` | Cosine similarity of two embeddings |
| `euclidean_distance(a, b)` | `RETURN euclidean_distance(n.emb, (0.4, 0.3))` | Euclidean distance of two embeddings |

**Aggregate functions** (RETURN only — *not* valid in WHERE / ORDER BY / SKIP / LIMIT; only valid with a single return item):

| Function | Example |
|----------|---------|
| `count(n)` / `count(*)` | `MATCH (n:Person) RETURN count(*)` |
| `avg(expr)` | `MATCH (n:Person) RETURN avg(n.age)` |

Aggregates may be combined within that single return item, e.g. `RETURN count(n) + avg(n.age)`.

## UNWIND and list literals

```cypher
UNWIND [1, 2, 3] AS x RETURN x          -- expand a list into one row per element
RETURN [1, 2, 3] AS nums                -- return a list literal directly
```

`UNWIND` accepts **literal lists only** — `UNWIND someVar`, `UNWIND $param`, and `UNWIND collect(...)` are not yet supported. List literals use brackets `[...]`; their elements must be literals.

## LOAD CSV

`LOAD CSV` streams rows from a CSV file (under the `data/` directory) as a reading statement:

```cypher
LOAD CSV "people.csv" WITH HEADERS AS row RETURN row.name, row.age   -- by column name
LOAD CSV "people.csv" AS row RETURN row[0], row[1]                    -- by 0-based index
```

Add `ON ERROR SKIP` to skip malformed rows (default is `ON ERROR FAIL`). All values come back as strings — wrap in `toInteger`/`toFloat` as needed. To build a graph, drive `CREATE` from it inside a change: `LOAD CSV "people.csv" WITH HEADERS AS row CREATE (:Person {name: row.name})`. (`LOAD CSV` followed by `MATCH` in one statement is not yet supported.)

## Data Types

| Type | Cypher Example | pandas column type |
|------|----------------|--------------------|
| String | `name: 'Alice'` | `string` |
| Integer | `age: 30` | `Int64` |
| Unsigned integer | — | `UInt64` |
| Boolean | `flag: true` | `boolean` |
| Double | `score: 3.14` | `float64` |
| Embedding | `emb: (1.2, 2.0, 0.0)` | array |

Strings accept single quotes, double quotes, or backticks. **Embedding literals use parentheses** `(1.2, 2.0, 0.0)` (min 2 items); `[...]` is a list literal, a different type. The `UInt64` dtype is used for unsigned-integer results.

## Gotchas

- Aggregate functions (`count`, `avg`) are valid only in RETURN — using them in WHERE/ORDER BY/SKIP/LIMIT errors with "Invalid use of aggregate expression in this context"
- Comma-separated patterns are cartesian products, not joins — always check if you intended a join (shared variable) instead
- Keywords are case-insensitive (`MATCH` == `match`), but label/type names are case-sensitive
