---
name: turingdb-parquet
description: Use when importing data from Parquet files into a TuringDB graph via the `turing-parquet` CLI (bundled with the `turingdb` pip package from version 1.32). Covers required file schema, CLI flags, and how to make the imported graph available to a running server.
---

# TuringDB: Parquet Import

The `turing-parquet` CLI builds a TuringDB graph from one or more **node** Parquet files and one or more **edge** Parquet files. It ships with the `turingdb` pip package from **version 1.32 onwards**.

## Step 1 — Ensure version ≥ 1.32

```bash
uv add 'turingdb>=1.32'
# or
pip install 'turingdb>=1.32'
```

Verify the binary is on PATH:

```bash
uv run turing-parquet -h
# or, inside an activated venv:
turing-parquet -h
```

If `turing-parquet` is missing after install, the installed `turingdb` is older than 1.32 — upgrade explicitly.

## Step 2 — Prepare the Parquet files

Both node and edge inputs may be sharded: repeat `-nodes` / `-edges` once per file.

**Node files** must contain:

| Column | Type | Notes |
|--------|------|-------|
| `id` | string | Unique node identifier, referenced by edges. |
| `label` | string | Becomes the TuringDB node label (e.g. `Person`, `Gene`). |
| `properties` | string (JSON) | All node properties as a JSON object. Column name overridable with `-props`. |

**Edge files** must contain:

| Column | Type | Notes |
|--------|------|-------|
| `from` | string | Source node `id`. |
| `to` | string | Target node `id`. |
| `relation` | string | Becomes the TuringDB edge type. Column name overridable with `-edgetype`. |
| `properties` | string (JSON) | Edge properties as a JSON object. Same column name as nodes. |

**How the JSON `properties` column is unpacked:**
- Scalar fields → node/edge properties.
- Nested objects → **sub-record nodes** linked to the parent by a `HAS_<FIELD>` edge.
- Arrays of objects → multiple sub-record nodes, one per element.
- Identical sub-records (same inferred label and `id`/`value`) are deduplicated automatically.

## Step 3 — Run the import

```bash
uv run turing-parquet \
    -nodes data/nodes.parquet \
    -edges data/edges.parquet \
    -out ./turingdb.out \
    -graph mygraph
```

### Flags

**Required:**

| Flag | Purpose |
|------|---------|
| `-nodes <path>` | Path to a node Parquet file. Repeatable for sharded inputs. |
| `-edges <path>` | Path to an edge Parquet file. Repeatable. |
| `-out <dir>` | TuringDB root directory to write into (default `./turingdb.out`). Created if absent; **existing graphs in the same directory are preserved**. |
| `-graph <name>` | Graph name to write inside `-out` (default `imported`). **If a graph with this name already exists in the directory, its subdirectory is wiped before writing.** |

**Optional:**

| Flag | Purpose |
|------|---------|
| `-props <col>` | Override the JSON-properties column name. Defaults to `properties` when present in every input; otherwise the tool prompts. |
| `-edgetype <col>` | Override the edge-type column name. Defaults to `relation` when present in any edge file; otherwise the tool prompts. |

**Sharded example:**

```bash
uv run turing-parquet \
    -nodes data/nodes_part1.parquet -nodes data/nodes_part2.parquet \
    -edges data/edges_part1.parquet -edges data/edges_part2.parquet \
    -out ./turingdb.out \
    -graph mygraph
```

## Step 4 — Make the graph available to the server

Copy the imported graph into the TuringDB home data directory (`~/.turing/graphs/`), then start (or restart) the server:

```bash
cp -r ./turingdb.out/graphs/mygraph ~/.turing/graphs/
uv run turingdb start -demon
```

## Step 5 — Verify the import

```python
from turingdb import TuringDB

client = TuringDB(host="http://localhost:6666")
client.load_graph("mygraph")
client.set_graph("mygraph")

print(client.query("CALL db.labels()"))
print(client.query("CALL db.edgeTypes()"))
print(client.query("CALL db.propertyTypes()"))
```

If sub-record nodes were generated from nested JSON, you will see them as extra labels and `HAS_<FIELD>` edge types in the schema output.

## Gotchas

- **Same-name overwrite:** running with an existing `-graph` name wipes that graph's subdirectory inside `-out` before writing. Other graphs in the same `-out` are untouched.
- **`turing-parquet` is a separate binary** from `turingdb`. Both ship in the same pip package (≥ 1.32), but the CLI commands are independent — `turing-parquet` only builds the on-disk graph; it does not start a server or connect to one.
- **JSON properties must be valid JSON strings**, not Parquet structs. The tool parses the column as JSON.
- **Repeat the flag, don't comma-separate:** `-nodes a.parquet -nodes b.parquet`, not `-nodes a.parquet,b.parquet`.
