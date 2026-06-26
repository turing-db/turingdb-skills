---
name: turingdb-importing
description: Use when building a TuringDB graph from an external file — JSONL (Neo4j APOC export), GML, Parquet (via the turing-parquet CLI), or migrating from Neo4j. Covers whole-graph import, not loading an existing TuringDB graph (see startup.md).
---

# TuringDB: Importing External Data

These commands build a **new** graph from an external file. To load an existing on-disk TuringDB graph instead, see `startup.md` (`client.load_graph(...)` / `LOAD GRAPH <name>`).

External files must live in the **`data/` subdirectory** of the TuringDB working directory (`-turing-dir`, default `~/.turing`). `LOAD JSONL` / `LOAD GML` are engine commands — run them via `client.query(...)`.

## LOAD JSONL

Imports a JSONL graph file (one JSON object per line). The expected format is **Neo4j APOC's JSON export** (`apoc.export.json.all` with `useTypes: true`).

```python
client.query("LOAD JSONL 'mydata.jsonl' AS mygraph")   # creates graph 'mygraph'
client.set_graph("mygraph")
```

Line format:

```json
{"type":"node","id":"0","labels":["Person"],"properties":{"name":"Alice","age":30}}
{"type":"relationship","id":"0","label":"KNOWS","start":{"id":"0"},"end":{"id":"1"},"properties":{"since":2021}}
```

- **Node** `id` must start at `0` and increment by `1` with no gaps; needs at least one label.
- **Relationship** needs exactly one `label`; `start`/`end` reference node `id`s (bare integer or `{"id": ...}`).
- Array-valued properties are stored as strings by default. To load them as `Embedding` properties instead, append `WITH EMBEDDINGS [{"emb", 384}, ...]` (dimension > 1) — see `algorithms.md`.

## LOAD GML

Imports a Graph Modeling Language file.

```python
client.query("LOAD GML 'mygraph.gml' AS mygraph")
client.set_graph("mygraph")
```

> **Caveat:** every imported node gets the label **`GMLNode`** and every edge the type **`GMLEdge`**, regardless of fields in the file. Any field other than `id`, `source`, `target` becomes a plain property (stored as a string). Query with `MATCH (n:GMLNode)` and `MATCH ()-[e:GMLEdge]->()`.

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

## Parquet — the `turing-parquet` CLI

Parquet import is a **separate command-line tool** (not a Cypher command). It reads node and edge Parquet files and writes a TuringDB graph to disk. `turing-parquet` ships with the `turingdb` package from **version 1.32** onward (`uv add 'turingdb>=1.32'`); if the binary is missing after install, upgrade.

```bash
turing-parquet \
    -nodes data/nodes.parquet \
    -edges data/edges.parquet \
    -out  ./turingdb.out \
    -graph mygraph
```

**Expected columns:**

- **Node files** (`-nodes`): `id` (string, unique), `label` (string), and a JSON-string properties column (default `properties`, override with `-props`).
- **Edge files** (`-edges`): `from`, `to` (string node ids), an edge-type column (default `relation`, override with `-edgetype`), and the JSON properties column.

Property values must be a **valid JSON string** (not a Parquet struct). The JSON is expanded: scalar fields become properties; nested objects become **sub-record nodes** linked by `HAS_<FIELD>` edges; arrays of objects become multiple sub-records. Identical sub-records (same inferred label + `id`/`value`) are deduplicated into one shared node.

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

This writes the graph under `<-out>/graphs/<-graph>`. Move that directory into your TuringDB working dir and load it (start or restart the server if needed — see "Starting a server" in `startup.md`):

```bash
cp -r ./turingdb.out/graphs/mygraph ~/.turing/graphs/
```

```python
client.load_graph("mygraph")
client.set_graph("mygraph")
```

**Gotchas:**
- `turing-parquet` is a separate binary from `turingdb` (same pip package, ≥ 1.32) — it only builds the on-disk graph; it does **not** start or connect to a server.
- Re-running with an existing `-graph` name **wipes** that graph's subdirectory inside `-out` first (other graphs in `-out` are left intact).
- The `properties` column must be a **valid JSON string**, not a Parquet struct.
- Repeat `-nodes`/`-edges` once per file for sharded inputs — don't comma-separate.

## After importing

Inspect what landed with the introspection procedures (see `introspection.md`):

```python
client.query("CALL db.labels()")
client.query("CALL db.edgeTypes()")
client.query("CALL db.propertyTypes()")
```

## Related

- **Streaming CSV rows** into a query (`LOAD CSV`) — see `querying.md`.
- **Embeddings** (`WITH EMBEDDINGS`, `LOAD EMBEDDING FROM`) — see `algorithms.md`.
- **Loading an existing TuringDB graph** (not an external file) — see `startup.md`.
