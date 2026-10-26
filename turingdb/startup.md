---
name: turingdb-startup
description: Use when connecting to TuringDB before querying — installs the package, starts/stops a TuringDB server, connects over HTTP (with optional auth token), or runs the engine in-process (embedded), then loads/creates a graph ready for exploration.
---

# TuringDB: Startup & Connection

By default, connect to a running TuringDB **server** over HTTP. The SDK uses this backend unless told otherwise, and the following all require it: the browser visualizer, on-disk graph discovery, concurrent multi-process access, and S3 transfers. If you want a self-contained engine with no server to manage, use the in-process (embedded) backend instead (see the last section).

## Step 1 — Ensure turingdb is installed

```bash
uv add 'turingdb>=3.0.2'
```

If the project doesn't use uv:

```bash
pip install 'turingdb>=3.0.2'
```

The wheel (Python 3.10–3.14; Linux x86_64/aarch64, macOS arm64) includes:
- the `turingdb` CLI (server and interactive shell),
- the `turing-parquet` import tool,
- the in-process engine,
- the bundled `greeter` example extension.

Pin `>=3.0.2`. That release adds the features this skill describes beyond 3.0: label/edge-type parameters, label expressions, string and math functions, `NEXT`, `WHEN`, `FOREACH` and `COLLECT {}`, among others. Older projects may pin `1.x`, which uses the previous engine and a much smaller Cypher subset.

**Client and server versions must match across the 3.0.2 boundary.** From 3.0.2, the SDK sends each query as a JSON body (see Raw HTTP below), and the server accepts nothing else. An older SDK against a 3.0.2 server gets `PARSE_ERROR: The request body must be a JSON object`, so upgrade both together.

## Step 2 — Connect to a server

Connect to a running TuringDB server — it may already be running. `host` is the full URL, not a host+port pair.

```python
from turingdb import TuringDB

client = TuringDB(host="http://localhost:6666")   # HTTP/JSON client (the default backend)
client.list_loaded_graphs()                        # raises if the server isn't reachable
```

If nothing is listening on that port, start a server first ("Starting a server" below), then connect.
- **`httpx` exceptions:** connection failures, and HTTP errors without a JSON body such as 401.
- **`TuringDBException`:** query errors, and HTTP errors the server explains in JSON, such as `REQUEST_TOO_BIG` (HTTP 413).

### Authentication

A server started with `-auth-on` requires a bearer token. Pass it with `token=`, or set the `TURINGDB_AUTH_TOKEN` env var, which the SDK reads automatically:

```python
client = TuringDB(host="http://localhost:6666", token="s3cret")
```

A missing or wrong token gets an HTTP 401 with an empty body, raised as `httpx.HTTPStatusError`. Raw HTTP clients send `Authorization: Bearer <token>`.

## Step 3 — Create or load a graph

A fresh server starts on the `default` graph. To work on a specific graph, create it or load an existing one by name:

```python
client.create_graph("my_graph")   # create new (errors if it already exists)
# or
client.load_graph("my_graph")     # load an existing on-disk graph by name
client.set_graph("my_graph")      # make it the active graph

print("Loaded:", client.list_loaded_graphs())
```

To discover on-disk graphs you haven't loaded yet, use `client.list_available_graphs()`. It is only available on the server's `json` backend, not in embedded mode. There is no command to drop or unload a graph.

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

Start a server with the `turingdb` CLI if one isn't already running. `turingdb` has `start` and `stop` subcommands (`start` is the default). **Flags use a single dash.**

| Flag | Meaning |
|------|---------|
| `-turing-dir <path>` | Root data directory (contains `graphs/`, `data/`, `vector/`, `logs/`). Default `~/.turing` |
| `-demon` | Run as a background daemon. Without it, `start` serves HTTP **and** opens an interactive shell |
| `-load <graph>` | Load a graph at startup (repeatable) |
| `-p <port>` | Listen port (default 6666) |
| `-i <addr>` | Listen address (default localhost) |
| `-in-memory` | Don't persist graphs to disk |
| `-reset-default` | Reset the content of the `default` graph |
| `-auth-on` | Require a bearer token. The token is read from `TURINGDB_AUTH_TOKEN`, and startup fails if that variable is unset |
| `-ui` / `-ui-port <port>` | Launch the browser visualizer (default port 8080) |
| `-no-vector-mmap` | Read HNSW vector indexes into memory at startup instead of memory-mapping the index files |

With `-demon`, `start` returns once the server is ready, or exits with status 1 and prints why it couldn't start, for example:
- `Could not start the server on 127.0.0.1:6666: bind: Address already in use`
- `TuringDB is already running (PID: …)` when the same `-turing-dir` is in use.

Try each invocation in order, stopping at the first that works:

```bash
uv run turingdb start -turing-dir <path> -demon     # uv project (recommended)
.venv/bin/turingdb start -turing-dir <path> -demon  # uv/standard venv
turingdb start -turing-dir <path> -demon            # activated/global install
```

- **Load a graph at startup:** add `-load <graph_name>` to the same command.
- **Visualizer:** add `-ui`, then open `http://localhost:8080`.
- **Authentication:** run `TURINGDB_AUTH_TOKEN=<secret> uv run turingdb start -demon -auth-on`.
- **Logs:** server logs go to `<turing-dir>/logs/turingdb.log`. Import errors such as `LOAD JSONL` failures give details only there.

### Stopping the server

Pass the **same `-turing-dir`** used at startup — otherwise `stop` looks at the default `~/.turing` and reports no instance found.

```bash
uv run turingdb stop -turing-dir <path>   # if started with -turing-dir
uv run turingdb stop                       # default data directory (~/.turing)
```

Add `-timeout <ms>` (alias `-t <ms>`) to change how long `stop` waits for the process to release its lock (default 3000).

### Interactive shell

`turingdb start` without `-demon` opens a Cypher shell on top of the server. Shell commands:
- `cd <graph>`: switch graph.
- `checkout change-<id>`, `checkout <commit>`, `checkout`: switch to a change, a commit, or back to HEAD.
- `:param <name> => <json value>`: set a query parameter (`:param lbl => "Person"`). `SHOW PARAMETERS` lists them, and `:param clear` removes them all.
- `read <file>`, `help`, `quit` / `exit`.

It is handy for manual exploration. Agents should use the SDK instead.

### Raw HTTP

The SDK is a thin client over one endpoint. The request body is a **JSON object** with the query and optional parameters:

```bash
curl -s -X POST 'http://localhost:6666/query?graph=my_graph' \
     -d '{"query": "MATCH (n:$lbl) RETURN count(n)", "parameters": {"lbl": "Person"}}'
```

- **Body:** `query` (string, required) and `parameters` (object, optional). A plain-text body fails with `PARSE_ERROR: The request body must be a JSON object`, and any other field with `PARSE_ERROR: Unknown field '<name>' in the request body`.
- **Optional query-string params:** `change=<decimal change id>` (as `CHANGE NEW` returns it) and `commit=<hash>`. A value that doesn't parse, or names a change or commit that doesn't exist, is an error, not a silent fallback to main.
- **Size limit:** the request line, headers and body together must fit in **1 MiB**. A bigger request gets HTTP 413 with `{"error":"REQUEST_TOO_BIG",…}`.
- **Success response:** `{"header":{"column_names":[...],"column_types":[...]},"data":[chunk,...],"time":ms}`, where each chunk is a list of columns.
- **Error response:** `{"error":"PARSE_ERROR|ANALYZE_ERROR|PLAN_ERROR|EXEC_ERROR|GRAPH_NOT_FOUND|CHANGE_NOT_FOUND|COMMIT_NOT_FOUND|...","error_details":"..."}`. The details name the missing graph, change or commit (e.g. `Change '99' does not exist`).

---

## Alternative: run in-process (embedded)

When you want a self-contained engine with no server to start, stop, or connect to, use the **embedded** backend — it runs the database in-process.

```python
from turingdb import TuringDB

client = TuringDB(type="embedded", data_dir="<path>")   # in-process, no server; omit data_dir for ~/.turing
```

- **`data_dir`** is the root data directory (holds `graphs/`, `data/`, …). Omit it to use the default `~/.turing`.
- **No server machinery:** there is no daemon, socket, port, readiness polling, or stop/cleanup.
- **Persistence:** writes still persist to disk on `CHANGE SUBMIT` (same `data_dir`), so work survives the process ending.

The embedded backend does **not** support the browser visualizer, `list_available_graphs()`, concurrent multi-process access, S3 transfers, or `token=`. Use a server for those.

## Native binary backend

`TuringDB(type="native", host="localhost", port=6666)` speaks TuringDB's binary protocol. It only works against a server started with the env var `USE_TURING_PROTO=1`. That server is then binary-only, so curl, the JSON client and the visualizer stop working against it. Pointed at a normal HTTP server, the native client errors or hangs. Stick with the default `json` backend unless you control the server and need the extra throughput.
