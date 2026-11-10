---
name: turingdb-writing
description: Use when creating or updating data in TuringDB v3.1 (to build a new graph beyond a few thousand entities, use LOAD PARQUET instead; see importing.md) — CREATE, MERGE, SET (incl. SET n = map / n += map), REMOVE, DELETE/DETACH DELETE, FOREACH, conditional writes, bulk writes with UNWIND over parameters, bulk edges between existing nodes by native ID pairs, LOAD CSV, property indexes, and the mandatory change/commit/submit workflow (including when COMMIT is required and how conflicts behave).
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
| `CREATE` / `MERGE` nodes or edges | **Not to MATCH.** A later `MATCH` finds nothing, and a `MATCH … CREATE` edge silently creates nothing. A later **`MERGE` does see them** and doesn't duplicate them |
| `SET` / `REMOVE` / `DELETE` on entities that already existed | Yes, immediately |
| anything, followed by `COMMIT` | Yes |
| writes earlier in the **same query**, including earlier `NEXT` segments | Yes |

**Rule of thumb: run `COMMIT` after any query that creates entities that a later query must MATCH.** Several COMMITs per change are fine. `CHANGE SUBMIT` commits whatever is still pending, so no COMMIT is needed right before it.

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

**Values from parameters:** pass the data, not query text: `client.query("CREATE (:Person {name: $name, age: $age})", {"name": "O'Brien", "age": 41})`, or a whole property map with `CREATE (n:Person) SET n = $props`.

**Rules:**
- **Labels and edge types:** every node needs at least one label, and every edge exactly one edge type. `[:A:B]` fails with "An edge in a write pattern takes exactly one edge type, with no '|', '&', ':' or '!'".
- **Label syntax:** labels in a write pattern can only be joined by `:` or `&` (`CREATE (n:A&B)`), not `|` or `!`. A label or an edge type can come from a parameter: `client.query("CREATE (n:$lbl {x: 1})", {"lbl": "Item"})`.
- **Labels are fixed at creation.** `SET n:Label` and `REMOVE n:Label` are not supported.
- **Property types are global.** Each property name has **one value type across the whole graph** (see `CALL db.propertyTypes()`). If `age` is an Integer, `CREATE (:X {age: 'old'})` and `SET n.age = 2.5` both fail with `types … are incompatible`. There is no Int → Double widening.
- **Nulls:** a null in a CREATE property map (`{name: null}`) is skipped, so the property is simply not set.
- **No variable-length edges in write patterns.** You also can't reuse a matched edge variable in CREATE.
- **Creating edges between known nodes:** address both ends by native node ID. One pair per query works (`MATCH (a) WHERE a = 12 MATCH (b) WHERE b = 34 CREATE …`). For many edges, batch the pairs in one UNWIND (see "Bulk edges between existing nodes" below).
- **Edges must be directed.** An undirected CREATE `(a)-[:R]-(b)` fails with "Only directed relationships are supported in CREATE". Parallel edges and self-loops are allowed.

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

- **Earlier writes:** MERGE matches nodes and edges written by earlier queries of the same change, even before `COMMIT`. Running the same MERGE twice creates one node.
- **Null keys:** a MERGE key that is null at run time is rejected ("MERGE cannot match on property 'k', which is null"). Filter nulls out first (`WHERE k IS NOT NULL`).
- **No constraints:** there are no uniqueness constraints, so MERGE is the way to avoid duplicates.

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
MATCH (n:Person {name: 'Alice'}) SET n += {city: 'Berlin', score: 1.5}             // add / update from a map
MATCH (c:Company {name: 'Acme'}) SET c = {name: 'Acme Corp', size: 10}             // replace all properties
UNWIND $rows AS row MATCH (n) WHERE n = row.id SET n += row.props                  // bulk update by native ID
```

- **`SET n += map`** adds and updates properties, and **`SET n = map`** replaces them all. A key set to `null` removes its property. The map can be a literal, a parameter or a map read from a row.

**Not supported:**
- `SET n:Label` ("SET cannot update entity types yet") and `REMOVE n:Label` ("Not implemented: REMOVE of a label").
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
- Deleting a path variable (`DELETE p`) is not supported ("Can only delete nodes or edges"). Delete its nodes or edges instead.

## FOREACH and conditional writes

`FOREACH` runs write clauses once per list element, without producing rows:

```cypher
FOREACH (i IN [1, 2, 3] | CREATE (:Num {v: i}))
FOREACH (nm IN ['x', 'y', 'x'] | MERGE (:Tag {name: nm}))                              // creates 2 nodes
MATCH (n:Person) WITH collect(n) AS ns FOREACH (x IN ns | SET x.touched = true)
FOREACH (i IN [1, 2] | FOREACH (j IN [10, 20] | CREATE (:Cell {v: i * j})))            // nested
```

`WHEN … THEN … ELSE` branches can write too (syntax in `querying.md`):

```cypher
WHEN true THEN CREATE (:A {v: 1}) ELSE CREATE (:B {v: 1})
MATCH (n:Person) CALL (n) { WHEN n.age > 30 THEN CREATE (:Senior {of: n.name}) } RETURN count(*)
MATCH (n:Person {name: 'Alice'}) RETURN n NEXT WHEN n.age > 18 THEN CREATE (:Adult {of: n.name}) ELSE CREATE (:Minor {of: n.name})
```

## Bulk writes

**Choose the method by what you are writing:**

Measured on TuringDB 3.1, on a 200k-node, 400k-edge graph (server-side time, then the client round trip):

| Writing… | Use | Measured |
|---|---|---|
| A **new graph** beyond roughly 5,000 nodes or 1,000 edges | `LOAD PARQUET` (see `importing.md`) | 200k nodes + 400k edges in **0.3 s** |
| **Edges between existing nodes** | `UNWIND $pairs` of native node ID pairs, 10,000–20,000 per query (below) | 20,000 edges in **3 ms** / 10 ms |
| **New nodes** | `UNWIND $rows AS row CREATE (…)`, 10,000–20,000 rows per query | 20,000 nodes in 4 ms / 60 ms |
| Generated nodes | `UNWIND range(1, 100000) AS i CREATE (:Num {v: i})` | 100k nodes in 3 ms |
| **Embeddings** for existing nodes | `LOAD EMBEDDING` (see `algorithms.md`) | 20,000 × 384-d in 0.1 s (3.0) |
| Updates by **native ID** | `UNWIND $rows AS row MATCH (n) WHERE n = row.id SET n += row.props` | 1,000 nodes in 1 ms |
| Upserts by **key property** | `UNWIND $rows AS row MERGE (n:P {pid: row.pid}) SET …` | 1,000 rows in 41 ms |
| Lookups by key property, one `MATCH` per row from a parameter list (`UNWIND $rows … MATCH (n:P {pid: row.pid})`) | **avoid**: map keys to native IDs, or use `MERGE` when the keys exist | 1,000 rows take 1.7–2.7 s: the label is scanned for every row, index or not |

The same data written into the query text as literals is slower to send and to parse: 10,000 ID pairs take 34 ms inlined against 3 ms as a parameter. Parameters also need no escaping.

`LOAD PARQUET` always creates a **new graph**; it cannot append to an existing one. To add a large volume of nodes to an existing graph, create them with `UNWIND $rows`, `COMMIT`, read back their native IDs, and add their edges by ID pairs.

### Bulk edges between existing nodes

Get the native node IDs once (`RETURN id(n)` alongside your business key), map your edge list to ID pairs in Python, and create the edges in batches:

```python
ids = client.query("MATCH (n:Person) RETURN n.pid AS pid, id(n) AS id")
key_to_id = dict(zip(ids["pid"].tolist(), ids["id"].tolist()))   # .tolist(): plain ints (numpy ints are refused as parameters)
pairs = [[key_to_id[s], key_to_id[t]] for s, t in edge_list]

client.new_change()
for i in range(0, len(pairs), 10_000):                       # stay well under the 1 MiB request limit
    client.query(
        "UNWIND $pairs AS p "
        "MATCH (a) WHERE a = p[0] MATCH (b) WHERE b = p[1] "
        "CREATE (a)-[:KNOWS]->(b)",
        {"pairs": pairs[i:i + 10_000]},
    )
client.query("CHANGE SUBMIT")
client.checkout()
```

- **Plan shape:** the plan is `unwind` → `fetch_nodes` × 2 → `create_edge`, with no scan. Check it with `EXPLAIN (db)` on a small batch.
- **Edge properties:** add them to each row: `[[12, 34, 2021], …]` with `CREATE (a)-[:KNOWS {since: p[2]}]->(b)`, or use map rows: `UNWIND $rows AS r MATCH (a) WHERE a = r.src MATCH (b) WHERE b = r.dst CREATE (a)-[:KNOWS {since: r.since}]->(b)`.
- **Request size:** the request, parameters included, must fit in 1 MiB, which is about 38,000 pairs of 7-digit IDs. If the SDK refuses a batch as too big, **create a new client**: the old one fails every later parameterized query (`Unknown CellTag.`, possibly a segfault), even after `reconnect()`.

### Small-volume patterns

Pass the data as a parameter and `UNWIND` it:

```python
client.query("UNWIND $names AS nm MERGE (:City {name: nm})", {"names": ["Brest", "Rennes", "Nantes"]})

rows = [{"name": "Brest", "pop": 139000}, {"name": "O'Brien Town", "pop": 1200}]
client.query("UNWIND $rows AS row CREATE (:City {name: row.name, pop: row.pop})", {"rows": rows})

# upsert by key: MERGE runs as one merge step (1,000 rows in ~40 ms); a per-row MATCH by key would scan
client.query("UNWIND $rows AS row MERGE (c:City {name: row.name}) SET c.pop = row.pop", {"rows": rows})
```

```cypher
UNWIND range(1, 1000) AS i CREATE (:Num {v: i})
UNWIND [['Brest', 139000], ['Rennes', 220000]] AS row CREATE (:City {name: row[0], pop: row[1]})
```

- **Values keep their types.** Values read from maps (`row.name`) and from mixed-type lists (`row[1]`) are written as they are. A new property takes the type of its values, and a value of the wrong type for an existing property fails with `Cannot set a property of type 'Int64' to a value of type 'String'`.
- **Never format values into the query** (f-strings, `%`, string concatenation): use parameters. Only `SKIP` / `LIMIT` and vector-search vectors still need literals.

**LOAD CSV**, for small files. The file must be under `<turing-dir>/data/`, and all fields are strings. Its key-property joins run as hash joins (1,000 edges in 12 ms on a 200k-node graph), but `LOAD PARQUET` is still the way to build a large graph:

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

Write-only `CALL { … }` subqueries work too: `UNWIND [1, 2] AS i CALL { WITH i CREATE (:X {v: i}) }`.

Keep each request under 1 MiB. To build a new graph, use `LOAD PARQUET` (see the table at the top of this section).

## Engine Commands for Change Management

These can be issued via `client.query()` or from the CLI:

| Command | Description |
|---------|-------------|
| `CHANGE NEW` | Create a new isolated change (returns `changeID`) |
| `CHANGE SUBMIT` | Commit pending writes and merge the current change into main |
| `CHANGE DELETE` | Discard the current change |
| `CHANGE LIST` | List the open change IDs for the current graph |
| `COMMIT` | Seal pending writes inside the change. Needed before later queries in the change can MATCH newly created entities |
| `MERGE_DATAPARTS` | Compact the graph's accumulated DataParts into one. Run on main with **no change open on any graph** |

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
- **Scoped indexes are not supported.** Label- or type-scoped indexes (`FOR (n:Person)`, `FOR [e:KNOWS]`) fail with "Constrained node indexes are not yet supported".
- **Neo4j syntax is not accepted.** `CREATE INDEX FOR (n:Person) ON (n.name)` and `IF NOT EXISTS` are rejected.
- **Listing:** use `CALL db.showIndexes()`. New indexes appear there after COMMIT/SUBMIT.
- **Constraints:** `CREATE CONSTRAINT` is not implemented.

## Gotchas

- Writes outside a change fail with "… outside of a write transaction". Always use the change workflow.
- Call `client.checkout()` after `CHANGE SUBMIT`, or the next query fails with `CHANGE_NOT_FOUND`.
- New entities are invisible to a later MATCH in the same change until `COMMIT`. The MATCH silently finds nothing. MERGE does see them.
- Labels can't be changed after creation. Nodes need at least one label, and edges exactly one type.
- Each property name has a single type graph-wide, and Integer properties don't widen to Double.
- Pass data as `$parameters`. Batches of 10,000–20,000 rows stay under the 1 MiB limit; after an oversized-parameter error, use a new client.
- There is no `SET n:Label`, `REMOVE n:Label` or constraint.
- For many edges between existing nodes, use native ID pairs. Don't match one key property per row from a parameter list.
