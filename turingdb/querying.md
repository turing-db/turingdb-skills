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
MATCH (a)-[e1]->(b)-[e2]->(c) RETURN a, c              -- multi-hop
```

## WHERE

```cypher
MATCH (n) WHERE n:Person RETURN n                           -- filter on label
MATCH (n) WHERE n.name = 'Alice' RETURN n
MATCH (n) WHERE n.age >= 18 AND n.city = 'London' RETURN n.name
MATCH (n) WHERE n.med = 'Aspirin' OR n.med = 'Ibuprofen' RETURN n.name
MATCH (n) WHERE n.institution IS NOT NULL RETURN n
MATCH (n)-[e]->(m) WHERE e:KNOWS RETURN n.name, m.name      -- filter on edge label
MATCH (n)-[e]->(m) WHERE e.since > 2020 RETURN n.name       -- filter on edge property
```

Operators: `=`, `<>`, `<`, `<=`, `>`, `>=`, `IS NULL`, `IS NOT NULL`

Inline property filter `{name: 'Alice'}` is exactly equivalent to `WHERE n.name = 'Alice'`.

## Joins vs Cartesian Products

```cypher
-- Cartesian product (N × M rows) — comma separates independent patterns:
MATCH (p:Person), (c:City) RETURN p.name, c.name

-- Join — shared variable links the patterns:
MATCH (a:Person)-->(i:Interest)<--(b:Person)
WHERE a.name <> b.name
RETURN a.name, b.name

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
```

## Built-in Functions

| Function | Example | Notes |
|----------|---------|-------|
| `labels(n)` | `RETURN labels(n), n.name` | Returns node label as string. **RETURN only — not usable in WHERE** |
| `toInteger(expr)` | `WHERE n.year > toInteger("2020")` | Parse string to int |
| `toFloat(expr)` | `RETURN n.price * toFloat("1.07")` | Parse string to float |

## Data Types

| Type | Cypher Example | pandas column type |
|------|----------------|--------------------|
| String | `name: 'Alice'` | `str` |
| Integer | `age: 30` | `int64` |
| Boolean | `flag: true` | `bool` |
| Double | `score: 3.14` | `float64` |
| Embedding | `emb: [1.2, 2.0, 0.0]` | array |

## Gotchas

- `labels(n)` cannot be used in WHERE — only in RETURN
- Comma-separated patterns are cartesian products, not joins — always check if you intended a join (shared variable) instead
