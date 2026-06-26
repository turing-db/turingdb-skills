---
name: turingdb-startup
description: Use when connecting to TuringDB before querying — installs the package, connects to a running TuringDB server (the default), or runs the engine in-process (embedded) for self-contained work, then loads/creates a graph ready for exploration.
---

# TuringDB: Startup & Connection

By default, connect to a running TuringDB **server** over HTTP — this is the backend the SDK uses unless told otherwise, and it's what the browser visualizer, on-disk graph discovery, concurrent multi-process access, and S3 transfers all require. If you want a self-contained engine with no server to manage, use the in-process (embedded) backend instead (see the last section).

## Step 1 — Ensure turingdb is installed

```bash
uv add turingdb
```

If the project doesn't use uv:

```bash
pip install turingdb
```

The wheel includes both the `turingdb` CLI (used to start a server) and the in-process engine.

## Step 2 — Connect to a server

Connect to a running TuringDB server — it may already be running. `host` is the full URL, not a host+port pair.

```python
from turingdb import TuringDB

client = TuringDB(host="http://localhost:6666")   # HTTP/JSON client (the default backend)
client.list_loaded_graphs()                        # raises if the server isn't reachable
```

If nothing is listening on that port, start a server first ("Starting a server" below), then connect.

## Step 3 — Create or load a graph

A fresh server starts on the `default` graph. To work on a specific graph, create it or load an existing one by name:

```python
client.create_graph("my_graph")   # create new
# or
client.load_graph("my_graph")     # load an existing on-disk graph by name
client.set_graph("my_graph")      # make it the active graph

print("Loaded:", client.list_loaded_graphs())
```

To discover on-disk graphs you haven't loaded yet, use `client.list_available_graphs()` (server/`json` backend only — not available in embedded mode).

## Step 4 — Explore the graph

Once a graph is loaded, read `introspection.md` (same directory) to map its shape:

```python
df_labels     = client.query("CALL db.labels()")
df_edge_types = client.query("CALL db.edgeTypes()")
df_props      = client.query("CALL db.propertyTypes()")
```

Print these before writing queries — they tell you what node labels, edge types, and properties exist.

---

## Starting a server

Start a server with the `turingdb` CLI if one isn't already running. `turingdb` has `start` and `stop` subcommands (`start` is the default). **Flags use a single dash**: `-turing-dir`, `-demon`, `-load`, `-p`, `-i`, `-in-memory`, `-ui`, `-ui-port`, `-reset-default`, `-start-timeout`.

| Flag | Meaning |
|------|---------|
| `-turing-dir <path>` | Root data directory (contains `graphs/`, `data/`) |
| `-demon` | Run as a background daemon |
| `-load <graph>` | Load a graph at startup (repeatable) |
| `-p <port>` | Override default port 6666 |
| `-in-memory` | Don't persist writes to disk |
| `-ui` / `-ui-port <port>` | Launch the browser visualizer (default port 8080) |
| `-start-timeout <ms>` | Time to wait for daemon readiness (default 500) |

Try each invocation in order, stopping at the first that works:

```bash
uv run turingdb start -turing-dir <path> -demon     # uv project (recommended)
.venv/bin/turingdb start -turing-dir <path> -demon  # uv/standard venv
turingdb start -turing-dir <path> -demon            # activated/global install
```

Load a graph at startup in the same command: add `-load <graph_name>`. Launch the visualizer with `-ui` (then open `http://localhost:8080`).

### Stopping the server

Pass the **same `-turing-dir`** used at startup — otherwise `stop` looks at the default `~/.turing` and reports no instance found.

```bash
uv run turingdb stop -turing-dir <path>   # if started with -turing-dir
uv run turingdb stop                       # default data directory (~/.turing)
```

Add `-timeout <ms>` (alias `-t <ms>`) to change how long `stop` waits for the process to release its lock (default 3000).

---

## Alternative: run in-process (embedded)

When you want a self-contained engine with no server to start, stop, or connect to, use the **embedded** backend — it runs the database in-process.

```python
from turingdb import TuringDB

client = TuringDB(type="embedded", data_dir="<path>")   # in-process, no server; omit data_dir for ~/.turing
```

`data_dir` is the root data directory (holds `graphs/`, `data/`, …); omit it to use the default `~/.turing`. There is no daemon, socket, port, readiness polling, or stop/cleanup. Writes still persist to disk on `CHANGE SUBMIT` (same `data_dir`), so work survives the process ending.

The embedded backend does **not** support the browser visualizer, `list_available_graphs()`, concurrent multi-process access, or S3 transfers — use a server for those.
