# TuringDB Skills for Claude Code

Claude Code skills that teach agents how to start, query, and manage [TuringDB](https://docs.turingdb.ai/) graph databases.

## What's included

| File | Covers |
|------|--------|
| `SKILL.md` | Entry point — routes to the right reference based on your task |
| `startup.md` | Install the package, start the server, connect, load a graph |
| `querying.md` | MATCH, WHERE, joins, ordering, functions |
| `writing.md` | CREATE, SET, and the change/commit workflow |
| `algorithms.md` | Shortest path (Dijkstra), vector/embedding search |
| `introspection.md` | Explore schema, versioning, time travel, SDK reference |

## Install

Copy the skill into your Claude Code skills directory:

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

- `/turingdb start the server at ~/mydata and load my_graph`
- `/turingdb query all Person nodes connected to Company nodes`
- `/turingdb add a new node with label Protein and name TP53`
- `/turingdb find the shortest path between two Station nodes`
- `/turingdb explore the schema of the loaded graph`

If the task spans multiple areas (e.g. start the server then query it), the skill reads multiple reference files in sequence.

## Uninstall

```bash
rm -rf ~/.claude/skills/turingdb
```
