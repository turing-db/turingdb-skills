---
name: turingdb-writing
description: Use when creating or updating data in TuringDB — CREATE nodes/edges, SET properties, and the mandatory change/commit workflow. All writes must go through a change or they will not persist.
---

# TuringDB: Writing Data

All writes (CREATE, SET) require the change workflow. TuringDB uses git-like branching — writes happen inside an isolated change that is then submitted to main.

## The Change Workflow

```python
change = client.new_change()       # create isolated change, returns integer ID
client.checkout(change=change)     # enter the change

# ... run your CREATE / SET queries ...

client.query("CHANGE SUBMIT")      # merge change into main
client.checkout()                  # return to main
```

Never run CREATE or SET outside a change — they will silently not persist.

## Complete End-to-End Example

```python
from turingdb import TuringDB, TuringDBException

client = TuringDB(host="http://localhost:6666")

# Set up graph
try:
    client.create_graph("demo")
except TuringDBException:
    pass
try:
    client.load_graph("demo")
except TuringDBException:
    pass
client.set_graph("demo")

# Write data
change = client.new_change()
client.checkout(change=change)

client.query("CREATE (:Person {name: 'Alice', age: 30})-[:KNOWS]->(:Person {name: 'Bob', age: 25})")
client.query("CREATE (:City {name: 'London'})")
client.query("COMMIT")  # persist so nodes are visible for next query

client.query("""
    MATCH (a:Person {name: 'Alice'}), (c:City {name: 'London'})
    CREATE (a)-[:LIVES_IN]->(c)
""")

client.query("CHANGE SUBMIT")
client.checkout()

# Verify
df = client.query("MATCH (n:Person)-[:LIVES_IN]->(c:City) RETURN n.name, c.name")
print(df)
#   n.name  c.name
# 0  Alice  London
```

## Creating Nodes and Edges

**Single query (preferred):** Create nodes and edges together — no intermediate COMMIT needed:

```python
client.query("CREATE (:Person {name: 'Jane'})-[:KNOWS]->(:Person {name: 'John'})")
client.query("CHANGE SUBMIT")
```

**Separate queries:** If you create nodes first and edges second, you must COMMIT after the nodes before creating edges:

```python
client.query("CREATE (:Person {name: 'Alice'}), (:Person {name: 'Bob'})")
client.query("COMMIT")   # persist nodes so they're visible for the next query
client.query("MATCH (a:Person {name: 'Alice'}), (b:Person {name: 'Bob'}) CREATE (a)-[:KNOWS]->(b)")
client.query("CHANGE SUBMIT")
```

## CREATE Syntax

```cypher
CREATE (:Person {name: 'Alice', age: 30})
CREATE (:Person {name: 'Mick'})-[:FRIEND_OF]->(:Person {name: 'John'})

-- Multiple nodes at once:
CREATE (:Person {name: 'Alice'}), (:Person {name: 'Bob'}), (:City {name: 'London'})

-- Connect two existing nodes:
MATCH (n:Person {name: 'Alice'}), (m:Person {name: 'Bob'})
CREATE (n)-[:KNOWS]->(m)

-- Create edge and return properties:
MATCH (n:Person {name: 'Alice'}), (m:Person {name: 'Bob'})
CREATE (n)-[:KNOWS]->(m)
RETURN n.name, m.name
```

Rules:
- Every node and edge requires at least one label
- Pure CREATE has no RETURN clause — only add RETURN when combining with MATCH

## SET Syntax (Update Properties)

SET always follows a MATCH. It creates the property if it doesn't already exist.

```cypher
MATCH (n:Person {name: 'Alice'}) SET n.age = 31

-- Multiple properties at once:
MATCH (n:Person {name: 'Alice'}) SET n.score = n.score * 1.1, n.updated = true

-- Expression-based update:
MATCH (p:Product) SET p.discountPrice = p.price * (1 - 0.15)

-- Update an edge property:
MATCH (n:Person {name: 'Alice'})-[e:KNOWS]->(m) SET e.since = 2020

-- Conditional update:
MATCH (n:Person) WHERE n.age < 18 SET n.isMinor = true

-- Set a vector embedding:
MATCH (n:Person {name: 'Alice'}) SET n.emb = [1.2, 2.0, 0.0, 12.0]
```

## LOAD CSV

Bulk-create nodes and/or edges from a CSV file, one row at a time. Must run inside a change like any other CREATE; the file must live in the `data` subdirectory (`~/.turing/data` by default), same as `LOAD GML` / `LOAD JSONL`.

**Positional access (no headers):**

```cypher
LOAD CSV 'mycsv.csv' AS row
CREATE (:NewNode {name: row[0], age: row[3], isFrench: row[2]})
```

**Named access (with headers):**

```cypher
LOAD CSV 'mycsv.csv' WITH HEADERS AS row
CREATE (:NewNode {name: row.names, age: row.ages, isFrench: row.isFrenches})
```

**Multi-pattern CREATE per row** — a single `LOAD CSV` can create several nodes and edges from the same row:

```cypher
LOAD CSV 'relationships.csv' WITH HEADERS AS row
CREATE (:Person {name: row.source})-[:KNOWS]->(:Person {name: row.target})
```

**Typing:** types are deduced automatically per cell — there is no explicit cast syntax. Numeric columns become `Int64` / `Double`, booleans become `Bool`, everything else `String`.

**Restrictions:**
- `LOAD CSV` + `MATCH` is **not currently supported** — you cannot match pre-existing nodes while streaming rows. To attach edges to nodes that already exist, load the rows into a staging label first, `COMMIT`, then run a separate `MATCH ... CREATE` query to wire them up.

## CREATE INDEX

TuringDB supports indexes on node and edge properties. Index DDL runs inside a change — the index becomes observable only after a subsequent `COMMIT` (within the change) or `CHANGE SUBMIT` (to main).

```cypher
CREATE INDEX personNameIdx FOR (n) ON n.name
CREATE INDEX knowsSinceIdx FOR [e] ON e.since
```

**Grammar:**
CREATE INDEX <index_name> FOR <pattern> ON <variable>.<propertyName>
where `<pattern>` is a bare node pattern `(n)` or a bare edge pattern `[e]`.

**Restrictions:**
- The pattern cannot yet be constrained by label or type — only `FOR (n)` and `FOR [e]` are accepted. `FOR (n:Person)` and `FOR [e:KNOWS]` are **not yet supported**.
- `DROP INDEX ... IF EXISTS` is **not supported**. To inspect the current index set use `CALL db.showIndexes()` (see `introspection.md`).

## Engine Commands for Change Management

These can be issued via `client.query()` or from the CLI:

| Command | Description |
|---------|-------------|
| `CHANGE NEW` | Create a new isolated change |
| `CHANGE SUBMIT` | Merge current change into main |
| `CHANGE DELETE` | Discard current change |
| `CHANGE LIST` | List active uncommitted changes |
| `COMMIT` | Persist intermediate state within a change (needed between separate node/edge creates) |

## Gotchas

- CREATE/SET outside a change do not persist — always use the change workflow
- Creating nodes and edges in separate queries requires `COMMIT` between the two steps so the first set of nodes are visible to the second query
- Pure CREATE has no RETURN clause — only MATCH+CREATE supports RETURN
- Every node and edge must have at least one label — `CREATE (n)` will error; use `CREATE (n:Label)`
- `CHANGE SUBMIT` merges the change into main. After submitting, call `client.checkout()` to return the SDK context to main
- `create_graph()` raises `TuringDBException` if the graph name already exists; `load_graph()` raises if already loaded. Wrap in try/except for idempotent scripts (see `startup.md`)
- `LOAD CSV` + `MATCH` is not supported. Load rows as nodes, `COMMIT`, then `MATCH ... CREATE` for edges to pre-existing nodes
- `CREATE INDEX` / `LOAD CSV` both require the change workflow. Indexes are only visible after `COMMIT` or `CHANGE SUBMIT`
- `CREATE INDEX` patterns cannot be label/type-constrained yet — use `(n)` / `[e]`, not `(n:Label)` / `[e:TYPE]`
