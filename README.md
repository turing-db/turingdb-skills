# TuringDB Skills for Claude Code

Claude Code skills that teach agents how to start, query, and manage [TuringDB](https://docs.turingdb.ai/) graph databases.

Targets **TuringDB 3.1** (`turingdb>=3.1` on PyPI, `turingdb@3.1` on npm). 3.1 speaks a binary protocol over HTTP/1.1, takes query parameters, and supports most of openCypher. The skill documents what works, what doesn't, where TuringDB differs from Neo4j Cypher, and which query shapes are fast, with timings measured on 3.1.

## What's included

| File | Covers |
|------|--------|
| `SKILL.md` | Entry point — key differences from Neo4j Cypher, then routes to the right reference based on your task |
| `startup.md` | Install the package, start/stop a server, connect from Python (server, auth token, embedded) or JavaScript/TypeScript (`TuringClient`), load/create a graph |
| `querying.md` | MATCH/OPTIONAL MATCH, label expressions, variable-length and quantified paths, WITH, NEXT, WHEN, aggregation, UNWIND, maps and map projection, subqueries, functions, points, parameters, LOAD CSV, result types, and use cases for reduce, allReduce, COLLECT, EXISTS/COUNT, CALL and NEXT |
| `writing.md` | CREATE, MERGE, SET (incl. from a map), REMOVE, DETACH DELETE, FOREACH, bulk writes with parameters (edges by native ID pairs), indexes, and the change/commit workflow |
| `importing.md` | Import external data — CSV, Parquet (`LOAD PARQUET`, `turing-parquet`), JSONL, GML, Neo4j migration |
| `algorithms.md` | Shortest path, path finding with variable-length patterns, vector/embedding search, geographic distances, GNN sampling |
| `introspection.md` | Explore schema, procedures, versioning, time travel, SDK reference |

## Install

The easiest method is using the skills CLI:

```bash
npx skills add https://github.com/turing-db/turingdb-skills
```

Alternatively, copy the skill manually into your Claude Code skills directory:

```bash
cp -r turingdb ~/.claude/skills/
```

## Verify

Start a Claude Code session and type:

```
/turingdb
```

You should see the routing table. The skill automatically reads the relevant sub-file based on what you ask — no need to specify which file to load.

## Usage

Just invoke `/turingdb` and describe what you want to do. Examples:

- `/turingdb connect to the local TuringDB server and load my_graph`
- `/turingdb query all Person nodes connected to Company nodes`
- `/turingdb add a new node with label Protein and name TP53`
- `/turingdb find the shortest path between two Station nodes`
- `/turingdb import people.csv and knows.csv into a new graph`
- `/turingdb explore the schema of the loaded graph`

If the task spans multiple areas (e.g. connect then query), the skill reads multiple reference files in sequence.

## Uninstall

```bash
rm -rf ~/.claude/skills/turingdb
```
