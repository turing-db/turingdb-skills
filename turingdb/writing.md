---
name: turingdb-writing
description: Use when creating or updating data in TuringDB v3 (for bulk loads beyond a few thousand entities, use LOAD PARQUET instead; see importing.md) — CREATE, MERGE, SET, REMOVE, DELETE/DETACH DELETE, bulk writes with UNWIND/LOAD CSV, property indexes, and the mandatory change/commit/submit workflow (including when COMMIT is required and how conflicts behave).
---

# TuringDB: Writing Data

All writes (CREATE, MERGE, SET, REMOVE, DELETE, CREATE/DROP INDEX, COMMIT) must run **inside a change**. TuringDB uses git-like branching: you write in an isolated change, then submit it to main. A write sent outside a change fails with an explicit error, e.g. `Cannot perform CREATE outside of a write transaction.`

## The Change Workflow

```python
client.new_change()                # CHANGE NEW — the client is now on this change

client.query("CREATE (:Person {name: 'Alice'})-[:KNOWS]->(:Person {name: 'Bob'})")
# ... more writes; COMMIT between queries when a later query must see new entities (below) ...

client.query("CHANGE SUBMIT")      # merge the change into main (commits pending writes itself)
client.checkout()                  # REQUIRED: return the client to main
```

- `new_change()` returns the change ID and switches the client onto that change. Calling `client.checkout(change=id)` afterwards is harmless but redundant.
- After `CHANGE SUBMIT` the client still points at the submitted change, so the next query fails with `CHANGE_NOT_FOUND` until you call `client.checkout()`.
- To abandon a change, run `client.query("CHANGE DELETE")` and then `client.checkout()`.
- `new_change()` raises if the client is already on a change or a commit. Call `checkout()` first.

### When you need `COMMIT` inside a change

`COMMIT` seals the pending writes of a change. It creates a commit in history but does not publish anything to main. The visibility rules are:

| Earlier query in the same change did… | Visible to later queries before `COMMIT`? |
|---|---|
| `CREATE` nodes/edges | **No.** A later `MATCH` finds nothing. A `MATCH … CREATE` edge silently creates nothing, and `MERGE` **creates duplicates** |
| `SET` / `REMOVE` / `DELETE` on entities that already existed | Yes, immediately, **except to `shortestPath`**: it does not see an uncommitted `SET` of its weight property (it does see a `DELETE`). Run `COMMIT` before routing |
| anything, followed by `COMMIT` | Yes |
| writes earlier in the **same query** | Yes |

**Rule of thumb: run `COMMIT` after any query that creates entities that a later query must MATCH or MERGE.** Several COMMITs per change are fine. `CHANGE SUBMIT` commits whatever is still pending, so no COMMIT is needed right before it.

```python
client.new_change()
client.query("CREATE (:Person {name: 'Alice'}), (:Person {name: 'Bob'})")
client.query("COMMIT")   # make the new nodes visible to the next query
client.query("MATCH (a:Person {name: 'Alice'}), (b:Person {name: 'Bob'}) CREATE (a)-[:KNOWS]->(b)")
client.query("CHANGE SUBMIT")
client.checkout()
```

### Failed queries and conflicts

- **A failed query can leave partial writes in the change.** After an `EXEC_ERROR` mid-change, the safest recovery is `CHANGE DELETE` and redoing the work.
- **Concurrent changes are isolated.** Main doesn't see a change's writes, even committed ones, until it is submitted. Each change rebases onto main at submit.
- **Disjoint edits submit cleanly.** If two changes edit the same entity, the second `CHANGE SUBMIT` fails. Example error: `This change attempted to update Node 0 (...) which has been modified on main.`
- **Conflict detection is coarse.** Creating an edge to a node that was modified on main also conflicts.
- **A failed submit rejects the whole change but leaves it open.** You can still fix it, or run `CHANGE DELETE`.
- **IDs of newly created nodes and edges are provisional until COMMIT/SUBMIT.** Read them after committing. IDs of existing nodes are stable across submits and restarts, so keep using them (only `MERGE_DATAPARTS` renumbers). See "Addressing nodes by native node ID" in `querying.md`.

## CREATE

```cypher
CREATE (:Person {name: 'Alice', age: 30})
CREATE (a:Person:Employee {name: 'Bob'}) RETURN a.name, id(a)                   // multiple labels; RETURN is allowed
CREATE (:Person {name: 'Mick'})-[:FRIEND_OF {since: 2020}]->(:Person {name: 'John'})
CREATE (a:T {x: 1}), (b:T {x: 2}), (a)-[:R]->(b)                                 // nodes and edges in one statement
CREATE p = (:City {name: 'Paris'})-[:ROAD {km: 450}]->(:City {name: 'Lyon'}) RETURN p
CREATE (c:City {name: 'Li' + 'lle', pop: 2 * 1000, tags: ['a', 'b'], info: {zone: 1}, emb: (0.1, 0.2, 0.3)})
CREATE (n:Item {sku: 'A1'}) SET n.price = 9.99 RETURN n.sku                      // CREATE then SET
CREATE (a:W {k: 'a'}) WITH a CREATE (b:W {k: 'b'})-[:NEXT]->(a)                  // WITH between writes

// Connect existing nodes (they must be committed or created earlier in this same query).
// If you know their native node IDs, address them directly (fastest; constants only, see note below):
MATCH (a) WHERE a = 12 MATCH (b) WHERE b = 34 CREATE (a)-[:KNOWS {since: 2021}]->(b)
// Otherwise by a key property:
MATCH (a:Person {name: 'Alice'}), (b:Person {name: 'Bob'})
CREATE (a)-[:KNOWS {since: 2021}]->(b)
RETURN a.name, b.name
```

**Rules:**
- **Labels and edge types:** every node needs at least one label, and every edge exactly one edge type. Two types (`[:A:B]`) is a parse error.
- **Labels are fixed at creation.** `SET n:Label` and `REMOVE n:Label` are not supported.
- **Property types are global.** Each property name has **one value type across the whole graph** (see `CALL db.propertyTypes()`). If `age` is an Integer, `CREATE (:X {age: 'old'})` and `SET n.age = 2.5` both fail with `types … are incompatible`. There is no Int → Double widening.
- **Nulls:** `{name: null}` is rejected in CREATE. Leave the key out instead.
- **No variable-length edges in write patterns.** You also can't reuse a matched edge variable in CREATE.
- **Creating edges between known nodes:** use their native node IDs as two **literal constants** per query (`MATCH (a) WHERE a = 12 MATCH (b) WHERE b = 34 CREATE …`). That is about 0.1 ms per edge server-side on a 1M-node graph. **Never** take both endpoints from one UNWIND row (`a = p[0]`, `b = p[1]`): that plans as a cross product of two full node scans and can run the server out of memory. For thousands of edges, use `LOAD PARQUET`.
- **Edges:** parallel edges and self-loops are allowed. An undirected CREATE `(a)-[:R]-(b)` is accepted and creates `a->b`.

## MERGE

MERGE matches the whole pattern or creates it, and supports `ON CREATE SET` / `ON MATCH SET`:

```cypher
MERGE (n:Person {name: 'Alice'})
  ON CREATE SET n.created = true
  ON MATCH SET n.seen = true
RETURN n.name

MATCH (a:Person {name: 'Alice'}), (b:Person {name: 'Carol'})
MERGE (a)-[e:KNOWS]->(b) ON CREATE SET e.since = 2024

MERGE (a:Tag {name: 'x'})-[:LINKS]->(b:Tag {name: 'y'})                  // whole pattern
MATCH (p:Person {name: 'Bob'}) MERGE (p)-[:OWNS]->(c:Car {model: 'Z'})    // bound start, new end
UNWIND ['a', 'b', 'a'] AS k MERGE (t:Key {k: k}) RETURN t.k              // dedups within one query
```

MERGE **cannot see nodes CREATEd by an earlier uncommitted query in the same change**, so it will duplicate them. Run `COMMIT` first. There are no uniqueness constraints, so MERGE is the way to avoid duplicates.

## SET and REMOVE (properties)

```cypher
MATCH (n:Person {name: 'Alice'}) SET n.age = 31
MATCH (n:Person {name: 'Alice'}) SET n.score = n.score * 1.1, n.updated = true      // several at once
MATCH (p:Product) SET p.discountPrice = p.price * (1 - 0.15)                       // expression
MATCH (a:Person) SET a.band = CASE WHEN a.age > 30 THEN 'senior' ELSE 'junior' END
MATCH (a:Person) SET a.nick = coalesce(a.nick, a.name)
MATCH (n:Person {name: 'Alice'})-[e:KNOWS]->(m) SET e.weight = 0.5                 // edge property
MATCH (n:Person) WHERE n.age < 18 SET n.isMinor = true                             // conditional
MATCH (n:Person {name: 'Alice'}) SET n.tags = ['a', 'b'], n.meta = {level: 3}      // list / map values
MATCH (n:Person {name: 'Alice'}) SET n.emb = (1.2, 2.0, 0.0, 12.0)                 // one-off Embedding; in bulk use LOAD EMBEDDING
MATCH (n:Person {name: 'Alice'}) SET n.joined = datetime('2024-03-15T10:30:00Z')
MATCH (a:Person) WITH count(a) AS c MATCH (s:Stats {name: 'people'}) SET s.total = c   // aggregate via WITH
MATCH (n) WHERE n = 1234 SET n.flag = true                                          // by native node ID: direct seek (fastest)
UNWIND [12, 34, 56] AS x MATCH (n) WHERE n = x SET n.flag = true                    // batch: literal, distinct node IDs
MATCH (n:Person {name: 'Alice'}) SET n.age = null                                  // clear a value
MATCH (n:Person {name: 'Alice'}) REMOVE n.score, n.flag                            // remove properties
```

**Not supported:**
- `SET n += {…}` fails with "SET cannot dynamically mutate properties yet".
- `SET n = {…}` is a parse error.
- `SET n:Label` and `REMOVE n:Label` fail.
- Aggregates directly in SET (`SET n.c = count(*)`). Compute them in a `WITH` first.

> **Embedding literals use parentheses `( )`**: `(1.2, 2.0, 0.0)`. Square brackets `[...]` are a list literal — a different type. Embedding literals need at least 2 numeric items.

## DELETE and DETACH DELETE

```cypher
MATCH (n:Person {name: 'Alice'}) DETACH DELETE n                  // node plus all its edges
MATCH (n:Person {name: 'Alice'})-[e:KNOWS]->(m) DELETE e          // one edge
MATCH (n:Temp) WHERE n.v > 3 DELETE n                             // nodes without edges
MATCH (n:Person {name: 'Remy'})-[e]->(m:Person) DELETE e, n       // several bound entities
```

- Plain `DELETE` of a node that still has edges fails with `Cannot delete a node with relationships; use DETACH DELETE`.
- Deleting a path variable (`DELETE p`) is not supported. Delete its nodes or edges instead.

## Bulk writes

> **Beyond a few thousand entities, use `LOAD PARQUET`, not Cypher writes.** Above roughly 5,000 nodes or 1,000 edges, write the data to Parquet and load it with `LOAD PARQUET` (see `importing.md`). Prefer it over every other write method: UNWIND batches, LOAD CSV + CREATE/MERGE, and chunked `client.query` loops.
>
> It matters most for **edges**. Creating an edge from Cypher has to MATCH both endpoints for every row, and that cost grows roughly quadratically. A property index on the key does not help. Measured on TuringDB 3.0:
>
> | Method | 1k nodes + 2k edges | 100k nodes + 200k edges |
> |---|---|---|
> | `LOAD PARQUET` | < 0.2 s | **0.2 s** |
> | `LOAD CSV` nodes, then `LOAD CSV` + MATCH + CREATE edges | 20 s | did not finish in 8 min |
> | `UNWIND` batches + MATCH + CREATE edges | 55 s | not attempted |
>
> Node-only creation through `UNWIND`/`LOAD CSV` is fast (100k nodes in 0.2 s). What makes Cypher bulk loads slow is MATCH-driven edge creation.
>
> **Embeddings for existing nodes** are the exception: write them back with `LOAD EMBEDDING`, which adds properties to the current graph (see `algorithms.md`). Don't use a `SET` loop.
>
> `LOAD PARQUET` always creates a **new graph**; it cannot append to an existing one. To add a large volume of data to an existing graph, rebuild it: write the existing data plus the new data to Parquet and `LOAD PARQUET` it under a new graph name. Use Cypher writes only for small, incremental changes.

The patterns below are for **small volumes** only. There are no parameters and no map field access (`row.x` on a map fails), so a Python list of dicts can't be passed in directly.

**UNWIND over inlined lists:**

```cypher
UNWIND range(1, 1000) AS i CREATE (:Num {v: i})
UNWIND ['Brest', 'Rennes', 'Nantes'] AS nm MERGE (:City {name: nm})
// Row of same-typed values:
UNWIND [['Brest', 'FR-29'], ['Rennes', 'FR-35']] AS row CREATE (:City {name: row[0], code: row[1]})
// Parallel lists for mixed types:
WITH ['Brest', 'Rennes'] AS names, [139000, 220000] AS pops
UNWIND range(0, size(names) - 1) AS i CREATE (:City {name: names[i], pop: pops[i]})
```

A CREATE can't read an element of a mixed-type list row (`['Brest', 139000]`). Use parallel lists instead.

**LOAD CSV**, for small files only (edges especially; see above). The file must be under `<turing-dir>/data/`, and all fields are strings:

```cypher
// 1. nodes
LOAD CSV 'people.csv' WITH HEADERS AS row
CREATE (:Person {pid: toInteger(row.id), name: row.name, age: toInteger(row.age)})
// 2. COMMIT  (so step 3 can MATCH the new nodes)
// 3. edges
LOAD CSV 'knows.csv' WITH HEADERS AS row
MATCH (a:Person {pid: toInteger(row.src)}), (b:Person {pid: toInteger(row.dst)})
CREATE (a)-[:KNOWS {since: toInteger(row.since)}]->(b)
```

Or do it in **one pass** with MERGE, which dedups across the rows of a single query, so no COMMIT is needed:

```cypher
LOAD CSV 'knows.csv' WITH HEADERS AS row
MERGE (a:Person {pid: toInteger(row.src)})
MERGE (b:Person {pid: toInteger(row.dst)})
CREATE (a)-[:KNOWS {since: toInteger(row.since)}]->(b)
```

`FOREACH` is **not** supported; use `UNWIND`. Write-only `CALL { … }` subqueries work: `UNWIND [1, 2] AS i CALL { WITH i CREATE (:X {v: i}) }`.

**Building the Cypher text in Python.** Escape quotes yourself, since there are no parameters:

```python
def lit(v):
    if isinstance(v, str):
        return "'" + v.replace("\\", "\\\\").replace("'", "\\'") + "'"
    if isinstance(v, bool):
        return "true" if v else "false"
    return repr(v)

names = ["Alice", "O'Brien"]
client.query(f"UNWIND [{', '.join(lit(n) for n in names)}] AS nm MERGE (:Person {{name: nm}})")
```

Keep each query small (at most a few thousand rows). For anything bigger, use `LOAD PARQUET` (see the note at the top of this section).

## Engine Commands for Change Management

These can be issued via `client.query()` or from the CLI:

| Command | Description |
|---------|-------------|
| `CHANGE NEW` | Create a new isolated change (returns `changeID`) |
| `CHANGE SUBMIT` | Commit pending writes and merge the current change into main |
| `CHANGE DELETE` | Discard the current change |
| `CHANGE LIST` | List the open change IDs for the current graph |
| `COMMIT` | Seal pending writes inside the change. Needed before later queries in the change can MATCH/MERGE newly created entities |
| `MERGE_DATAPARTS` | Compact the graph's accumulated DataParts into one. Run on main with **no change open on any graph**. It renumbers node IDs, and it is **not persisted until another change is submitted**: a restart before that silently reverts it |

## Property Indexes

A property index speeds up equality lookups (`WHERE n.prop = <value>`), because the optimizer rewrites them into an index scan. Creating or dropping an index is a **write**, so do it inside a change:

```cypher
CREATE INDEX age_index FOR (n) ON n.age        // node property index
CREATE INDEX since_index FOR [e] ON e.since    // edge property index (square brackets)
DROP INDEX age_index
```

```python
client.new_change()
client.query("CREATE INDEX age_index FOR (n) ON n.age")
client.query("CHANGE SUBMIT")
client.checkout()
```

**Rules:**
- **The property must already exist** in the graph.
- **The index name is required.**
- **Scoped indexes are not supported.** Label- or type-scoped indexes (`FOR (n:Person)`, `FOR [e:KNOWS]`) fail with "not yet supported".
- **Neo4j syntax is not accepted.** `CREATE INDEX FOR (n:Person) ON (n.name)` and `IF NOT EXISTS` are rejected.
- **Listing:** use `CALL db.showIndexes()`. New indexes appear there after COMMIT/SUBMIT.
- **Constraints:** `CREATE CONSTRAINT` is not implemented.

## Gotchas

- Writes outside a change fail with "… outside of a write transaction". Always use the change workflow.
- Call `client.checkout()` after `CHANGE SUBMIT`, or the next query fails with `CHANGE_NOT_FOUND`.
- New entities are invisible to later queries in the same change until `COMMIT`. A MATCH silently finds nothing, and a MERGE duplicates.
- Labels can't be changed after creation. Nodes need at least one label, and edges exactly one type.
- Each property name has a single type graph-wide.
- There are no `$parameters`, `SET +=`, FOREACH, or constraints.
