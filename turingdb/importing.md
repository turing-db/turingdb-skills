---
name: turingdb-importing
description: Use when building a TuringDB graph from external files — CSV (LOAD CSV), Parquet (LOAD PARQUET or the turing-parquet CLI), JSONL (Neo4j APOC export), GML, or migrating from Neo4j. Covers whole-graph import, not loading an existing TuringDB graph (see startup.md).
---

# TuringDB: Importing External Data

These commands build graph data from external files. To load an existing on-disk TuringDB graph instead, see `startup.md` (`client.load_graph(...)` / `LOAD GRAPH <name>`).

External files must live in the **`data/` subdirectory** of the TuringDB working directory (`-turing-dir`, default `~/.turing`). Paths in these commands are resolved inside `data/`; even an absolute path is re-rooted there. All of the `LOAD …` commands are engine commands, so run them via `client.query(...)`. When an import fails, the details are written to `<turing-dir>/logs/turingdb.log`.

> **Choosing a method by volume:** above roughly 5,000 nodes or 1,000 edges, **use `LOAD PARQUET`**, converting the source to Parquet first if needed. It loads 100k nodes + 200k edges in about 0.2 s. Cypher-driven loads (LOAD CSV/UNWIND + MATCH + CREATE) need minutes for a few thousand edges and do not finish at that size. Convert CSV, JSON or DataFrames to the `__id`/`__labels` Parquet schema with pandas/pyarrow rather than streaming them through Cypher. Keep `LOAD CSV` for small files and incremental additions to an existing graph.

| Source | Command | Creates |
|--------|---------|---------|
| CSV files (small) | `LOAD CSV` + `CREATE`/`MERGE` inside a change | data in the current graph |
| Parquet (`__id`/`__labels` schema) | `LOAD PARQUET 'dir' AS g` | a new graph |
| Parquet (string ids + JSON properties) | `turing-parquet` CLI | an on-disk graph to `LOAD GRAPH` |
| Neo4j APOC JSON export | `LOAD JSONL 'f.jsonl' AS g` | a new graph |
| GML | `LOAD GML 'f.gml' AS g` | a new graph |

## CSV — LOAD CSV (small volumes)

Use `LOAD CSV` to drive `CREATE`/`MERGE` into the current graph, inside a change. This is for small files and for adding to an existing graph. For bulk loads, convert the CSV to Parquet and use `LOAD PARQUET` (below):

```python
client.new_change()
client.query("""
LOAD CSV 'people.csv' WITH HEADERS AS row
CREATE (:Person {pid: toInteger(row.id), name: row.name, age: toInteger(row.age)})
""")
client.query("COMMIT")   # make the new nodes visible to the edge pass
client.query("""
LOAD CSV 'knows.csv' WITH HEADERS AS row
MATCH (a:Person {pid: toInteger(row.src)}), (b:Person {pid: toInteger(row.dst)})
CREATE (a)-[:KNOWS {since: toInteger(row.since)}]->(b)
""")
client.query("CHANGE SUBMIT")
client.checkout()
```

- **Strings only:** all fields arrive as strings. Convert them with `toInteger` / `toFloat` / `toBoolean` / `datetime`.
- **Format limits:** comma-delimited only, with no `FIELDTERMINATOR` and no `FROM 'file:///…'`.
- **Malformed rows:** add `ON ERROR SKIP` to skip them.

For more patterns, such as a single-pass MERGE, see `writing.md`. For reading CSV without writing, see `querying.md`.

## Parquet — LOAD PARQUET (in-engine) — preferred for bulk data

`LOAD PARQUET` is the fastest way to get data into TuringDB, and the default for anything beyond a few thousand entities. It builds a new graph from a **directory under `data/`** containing `nodes.parquet` and `edges.parquet`:

```python
client.query("LOAD PARQUET 'mydir' AS mygraph")                       # graph name defaults to the directory name
client.query("LOAD PARQUET 'mydir' AS mygraph WITH DURATIONS [tenure]")  # int64 µs columns → Duration
client.set_graph("mygraph")
```

**Required columns:**
- `nodes.parquet`: `__id` (int64, unique) and `__labels` (list<string>, non-empty).
- `edges.parquet`: `__source` (int64), `__target` (int64) and `__type` (string).

Every other column becomes a property:

| Parquet type | Property type |
|---|---|
| `int64` | Integer |
| `double` | Double |
| `bool` | Boolean |
| `string` | String |
| timestamp | DateTime |
| list of the types above | List |

Nulls are allowed. **`int32` and `float32` columns are rejected** ("Unsupported column type"), so cast them to int64 / float64 first.

- **New graph only:** the target graph name must not already exist. To grow an existing graph by a large amount, rebuild it from Parquet under a new name.
- **IDs:** `__id` values are your own int64 keys. Edges reference them through `__source`/`__target`.

**Recipe: pandas DataFrames → `LOAD PARQUET`.** Use this for CSV, JSON, SQL extracts or API data. Load them into pandas, map natural keys to int64 `__id`s, and write the two files:

```python
import pandas as pd, pyarrow as pa, pyarrow.parquet as pq
from pathlib import Path

people = pd.read_csv("people.csv")   # columns: email, name, age
knows  = pd.read_csv("knows.csv")    # columns: src, dst, since  (src/dst are emails)

ids = {key: i for i, key in enumerate(people["email"])}          # natural key -> int64 __id
nodes = pd.DataFrame({
    "__id": pd.Series(range(len(people)), dtype="int64"),
    "__labels": [["Person"]] * len(people),
    "email": people["email"],
    "name": people["name"],
    "age": people["age"].astype("Int64"),                          # int64 (nullable), never int32/float32
})
edges = pd.DataFrame({
    "__source": knows["src"].map(ids).astype("int64"),
    "__target": knows["dst"].map(ids).astype("int64"),
    "__type": "KNOWS",
    "since": knows["since"].astype("int64"),
})

out = Path.home() / ".turing" / "data" / "people_graph"           # <turing-dir>/data/<dir>
out.mkdir(parents=True, exist_ok=True)
pq.write_table(pa.Table.from_pandas(nodes, preserve_index=False), out / "nodes.parquet")
pq.write_table(pa.Table.from_pandas(edges, preserve_index=False), out / "edges.parquet")

client.query("LOAD PARQUET 'people_graph' AS people")
client.set_graph("people")
```

For several node types, concatenate them into one `nodes.parquet` with non-overlapping `__id` ranges, each row carrying its own `__labels`. Keep the natural key (`email` above) as a property so later queries can match on it.

## Parquet — the `turing-parquet` CLI

`turing-parquet` is a separate command-line tool that ships in the `turingdb` package. Use it when your tables have **string ids and a JSON-string properties column**. It reads node and edge Parquet files and writes a TuringDB graph to disk.

```bash
turing-parquet \
    -nodes data/nodes.parquet \
    -edges data/edges.parquet \
    -out  ./turingdb.out \
    -graph mygraph
```

**Expected columns:**

- **Node files** (`-nodes`): `id` (string, unique), `label` (string), and a JSON-string properties column (default `properties`, override with `-props`). The original string id is kept as property `id`.
- **Edge files** (`-edges`): `from`, `to` (string node ids), an edge-type column (default `relation`, override with `-edgetype`), and the JSON properties column.

**How the JSON is expanded:** property values must be a **valid JSON string**, not a Parquet struct.
- Scalar fields become properties.
- Nested objects become **sub-record nodes**, linked by `HAS_<FIELD>` edges.
- Arrays of objects become multiple sub-records.
- Identical sub-records (same inferred label + `id`/`value`) are deduplicated into one shared node.

**Flags:**

| Flag | Notes |
|------|-------|
| `-nodes` / `-edges` | Input files. **Repeatable** for sharded inputs — repeat the flag (`-nodes a -nodes b`), don't comma-separate. |
| `-out` | TuringDB root to write into (default `./turingdb.out`). Other graphs already in it are preserved. |
| `-graph` | Graph name inside `-out` (default `imported`). ⚠️ If that name already exists there, its subdirectory is **wiped** before writing. |
| `-props` / `-edgetype` | Override the properties / edge-type column names. Auto-detected when unambiguous; the tool prompts otherwise. |

For sharded inputs, repeat the flag once per file:

```bash
turing-parquet \
    -nodes data/nodes_part1.parquet -nodes data/nodes_part2.parquet \
    -edges data/edges_part1.parquet -edges data/edges_part2.parquet \
    -out ./turingdb.out -graph mygraph
```

This writes the graph under `<-out>/graphs/<-graph>`. Copy that directory into your TuringDB working dir, then load it. No server restart is needed:

```bash
cp -r ./turingdb.out/graphs/mygraph ~/.turing/graphs/
```

```python
client.load_graph("mygraph")
client.set_graph("mygraph")
```

**Gotchas:**
- `turing-parquet` only builds the on-disk graph. It does **not** start or connect to a server.
- Re-running with an existing `-graph` name **wipes** that graph's subdirectory inside `-out` first (other graphs in `-out` are left intact).
- The `properties` column must be a **valid JSON string**, not a Parquet struct.
- Repeat `-nodes`/`-edges` once per file for sharded inputs — don't comma-separate.

## LOAD JSONL

Imports a JSONL graph file (one JSON object per line). The expected format is **Neo4j APOC's JSON export** (`apoc.export.json.all` with `useTypes: true`).

```python
client.query("LOAD JSONL 'mydata.jsonl' AS mygraph")   # creates graph 'mygraph' (default name: file stem)
client.set_graph("mygraph")
```

Line format:

```json
{"type":"node","id":"0","labels":["Person"],"properties":{"name":"Alice","age":30,"tags":["a","b"]}}
{"type":"relationship","id":"0","label":"KNOWS","start":{"id":"0"},"end":{"id":"1"},"properties":{"since":2021}}
```

- **Node ids:** must be numeric (an int or a digit string). Gaps are fine. Every node needs at least one label.
- **Relationships:** each needs exactly one `label`. `start`/`end` reference node `id`s. Use the APOC object form `{"id": "..."}`. A bare integer `"end": 5` only works when node IDs have no gaps; otherwise the import fails with `unordered_map::at`.
- **Property values:** arrays load as `List` properties. Nested objects are stored as JSON strings. ISO date strings stay strings unless declared (see below).
- **Omit null properties; don't write `"x": null`.** A JSON `null` registers an extra String property named `x (String)` holding the text `"null"` next to the real `x`, on nodes and on edges, and it shows up in `db.propertyTypes()`. `LOAD PARQUET` handles nulls cleanly.
- **One `NaN` rejects the whole file** (Python's `json.dumps` writes a bare `NaN` by default; pass `allow_nan=False`). The error response names only the file. The line and column are in `<turing-dir>/logs/turingdb.log`.

Typed options can be chained after the graph name:

```cypher
LOAD JSONL 'mydata.jsonl' AS mygraph
  WITH EMBEDDINGS [{"emb", 384}, {"summary_vec", 768}]   // array → Embedding (dimension > 1)
  WITH DATETIMES [born, updatedAt]                      // ISO string → DateTime
  WITH DURATIONS [tenure]                               // integer microseconds → Duration
```

## LOAD GML

Imports a Graph Modeling Language file.

```python
client.query("LOAD GML 'mygraph.gml' AS mygraph")
client.set_graph("mygraph")
```

> **Caveats:**
> - Every imported node gets the label **`GMLNode`** and every edge the type **`GMLEdge`**, regardless of the fields in the file.
> - Any field other than `id`, `source` and `target` becomes a string property, and its **name gets a type suffix**, e.g. `label (String)`, `weight (String)`.
> - Query them with backticks: ``MATCH (n:GMLNode) RETURN n.`label (String)` ``. Run `CALL db.propertyTypes()` to see the exact names.

## Migrating from Neo4j

Export the Neo4j graph as JSONL with APOC, then `LOAD JSONL` it.

```cypher
// In Neo4j (APOC plugin required):
CALL apoc.export.json.all("output.json", {useTypes: true});
```

Copy the result into TuringDB's `data/` directory, then:

```python
client.query("LOAD JSONL 'output.json' AS mygraph")
```

(Neo4j 4.x databases must be migrated to the 5.x format with `neo4j-admin database migrate` before exporting.)

Most Neo4j read queries port directly to TuringDB v3. Review them for the differences listed in `SKILL.md`: no parameters, no string/math functions, no map projection, and the different shortest-path syntax.

## After importing

Inspect what landed with the introspection procedures (see `introspection.md`):

```python
client.query("CALL db.labels()")
client.query("CALL db.edgeTypes()")
client.query("CALL db.propertyTypes()")
```

## Related

- **Reading CSV rows** in a query (`LOAD CSV … RETURN`): see `querying.md`.
- **Embeddings** (`WITH EMBEDDINGS`, `LOAD EMBEDDING FROM`, vector indexes): see `algorithms.md`.
- **Loading an existing TuringDB graph** (not an external file): see `startup.md`.
