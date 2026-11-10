---
name: turingdb-startup
description: Use when connecting to TuringDB before querying — installs the package, starts/stops a TuringDB server, connects from Python over the binary protocol (with optional auth token) or runs the engine in-process (embedded), connects from JavaScript/TypeScript with the turingdb npm package, then loads/creates a graph ready for exploration.
---

# TuringDB: Startup & Connection

By default, connect to a running TuringDB **server**. The SDK uses this backend unless told otherwise, and the following all require it: the browser visualizer, concurrent multi-process access, S3 transfers and the JavaScript client. If you want a self-contained engine with no server to manage, use the in-process (embedded) backend instead (see the last section).

## Step 1 — Ensure turingdb is installed

```bash
uv add 'turingdb>=3.1'
```

If the project doesn't use uv:

```bash
pip install 'turingdb>=3.1'
```

The wheel (Python 3.10–3.14; Linux x86_64/aarch64, macOS 15+ arm64) includes:
- the `turingdb` CLI (server and interactive shell),
- the `turing-parquet` import tool,
- the in-process engine,
- the bundled `greeter` example extension.

Pin `>=3.1`. It adds value parameters, the binary protocol by default, map projection, points, `allReduce` and more over 3.0.2. Older projects may pin `1.x`, which uses the previous engine and a much smaller Cypher subset.

**Client and server versions must match across the 3.1 boundary.** A 3.1 client speaks the binary protocol of a 3.1 server, so upgrade the SDKs and the server together, then restart the server. A mismatch shows up as:
- `httpx.HTTPStatusError: Client error '400 Bad Request' …`: a Python SDK older than 3.1 against a 3.1 server.
- `Proto header dataLen does not match chunk payload size`: a 3.1 Python SDK against an older server.
- `PROTOCOL_ERROR: A packet payload of … bytes is more than the 67108864 bytes …`: a 3.1 JavaScript SDK against an older server.

## Step 2 — Connect to a server

Connect to a running TuringDB server — it may already be running. `host` is a host name or an IP address and `port` a number. A URL is refused: `host must be a host name or IP address, not a URL`.

```python
from turingdb import TuringDB

client = TuringDB(host="localhost", port=6666)   # same as TuringDB()
```

The client connects as soon as it is created. If nothing is listening, it raises `TuringDBException: Failed to connect to localhost:6666: Connection refused`; start a server first ("Starting a server" below), then connect.

Every failure raises `TuringDBException`, query errors and connection errors alike (the SDK no longer uses `httpx`).

### Authentication

A server started with `-auth-on` requires a bearer token. Pass it with `token=`, or set the `TURINGDB_AUTH_TOKEN` env var, which the SDK reads automatically:

```python
client = TuringDB(host="localhost", port=6666, token="s3cret")
```

A missing or wrong token raises `TuringDBException: Protocol error from server: Unauthorized: missing or invalid authentication token`.

## Step 3 — Create or load a graph

A fresh server starts on the `default` graph. To work on a specific graph, create it or load an existing one by name:

```python
client.create_graph("my_graph")   # create new (errors if it already exists)
# or
client.load_graph("my_graph")     # load an existing on-disk graph by name
client.set_graph("my_graph")      # make it the active graph

print("Loaded:", client.list_loaded_graphs())
```

To discover on-disk graphs you haven't loaded yet, use `client.list_available_graphs()` (any backend). There is no command to drop or unload a graph.

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
| `-demon` | Run as a background daemon. Without it, `start` serves queries **and** opens an interactive shell |
| `-load <graph>` | Load a graph at startup (repeatable) |
| `-p <port>` | Listen port (default 6666). Use another port, e.g. 6686, for Node.js clients (see below) |
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
- `:param <name> => <Cypher expression>`: set a query parameter (`:param lbl => 'Person'`, `:param ids => [0, 1, 2]`). `SHOW PARAMETERS` lists them, and `:param clear` removes them all.
- `read <file>`, `help`, `quit` / `exit`.

It is handy for manual exploration. Agents should use the SDK instead.

### The wire protocol

The server speaks only TuringDB's **binary protocol over HTTP/1.1**; the JSON API of 3.0 was removed, so `curl` and other plain HTTP tools can't run queries. A query is one `POST /query?graph=<g>[&change=<id>][&commit=<hash>]` with `Content-Type: application/turing-proto`, and the result streams back as binary packets, one per chunk, on a keep-alive connection. Use the Python or JavaScript client rather than speaking it yourself.

- **Size limit:** a request (query text plus parameters) must fit in **1 MiB**.
  - **Request too big:** the server refuses it with `Protocol error from server: …` and closes the connection. Call `client.reconnect()` and `set_graph()` again.
  - **Parameters over the limit:** the Python SDK refuses them before sending, with `The query parameters exceed 1048576 bytes`. That client is then broken for parameterized queries (see `SKILL.md`), so create a new one.
- **Change and commit:** a `change` or `commit` that doesn't exist is an error (`CHANGE_NOT_FOUND: Change '99' does not exist`), never a silent fallback to main.

---

## JavaScript and TypeScript: the `turingdb` npm package

`npm install turingdb@3.1` (match the server version). It runs in Node.js 18+ and in browsers, ships TypeScript declarations, and decodes results with a bundled WebAssembly module.

```js
import { TuringClient, TuringQueryError } from "turingdb";

const client = new TuringClient({
    url: "http://localhost:6686/query",   // http://<host>:<port>/query; default "/query"
    graph: "people",                      // default "default"
    // authToken: "s3cret",               // for a server started with -auth-on
});

// plain arrays: chunks → columns → rows; 64-bit integers as number
const [[names, ages]] = await client.queryData(
    "MATCH (p:Person) WHERE p.age >= $age RETURN p.name, p.age",
    { parameters: { age: 30 } },
);

// typed columns and the server time
const { chunks, execTimeMs } = await client.query("MATCH (n) RETURN count(n)");
```

- **Node.js can't reach port 6666.** Its `fetch` blocks 6666 as an unsafe port, so start the server on another one: `turingdb -demon -p 6686`.
- **Browsers:** the server sends no CORS headers. Serve `/query` from the page's own origin through a proxy to the server (Vite `server.proxy`, nginx `proxy_pass`) and leave `url` at its default `/query`. With Vite, add `optimizeDeps: { exclude: ["turingdb"] }` so the client stays next to its `.wasm` file. Let the proxy add the auth token rather than putting it in page code.
- **Options per query:** `parameters`, `graph`, `change`, `commit`, `signal` (an `AbortSignal`).
- **Client methods:** `setGraph` / `getGraph`, `setChange(id | "main")`, `setCommit(hash | "head")` and `setAuthToken`.
- **Writes:** run them inside a change, as in Python. Start one with `CHANGE NEW`, take the ID from the result, call `setChange(id)`, then write, `CHANGE SUBMIT` and `setChange("main")`.
- **Errors:** query errors reject with a `TuringQueryError` whose `status` is the code (`PARSE_ERROR`, …) and whose `message` points at the error.

Each query is a separate HTTP request, and creating a client opens no connection.

---

## Alternative: run in-process (embedded)

When you want a self-contained engine with no server to start, stop, or connect to, use the **embedded** backend — it runs the database in-process.

```python
import os
from turingdb import TuringDB

client = TuringDB(type="embedded", data_dir=os.path.expanduser("~/.turing"))   # omit data_dir for ~/.turing
```

- **`data_dir`** is the root data directory (holds `graphs/`, `data/`, …). Omit it to use the default `~/.turing`. Pass an **absolute path**: a leading `~` is not expanded.
- **No server machinery:** there is no daemon, socket, port, readiness polling, or stop/cleanup.
- **Persistence:** writes still persist to disk on `CHANGE SUBMIT` (same `data_dir`), so work survives the process ending.
- **Same API:** queries, parameters and DataFrames behave as on a server.

The embedded backend does **not** support the browser visualizer, concurrent multi-process access, S3 transfers, `token=` or `host=`. Use a server for those.
