---
name: turingdb-writing
description: Use when creating or updating data in TuringDB — CREATE nodes/edges, SET properties, and the mandatory change/commit workflow. All writes must go through a change or they will not persist.
---

# TuringDB: Writing Data

All writes (CREATE, SET) require the change workflow. TuringDB uses git-like branching — writes happen inside an isolated change that is then submitted to main.

## The Change Workflow

```python
change = client.new_change()       # create isolated change, returns integer ID
client.checkout(change=change)     # enter the change

# ... run your CREATE / SET queries ...

client.query("CHANGE SUBMIT")      # merge change into main
client.checkout()                  # return to main
```

Never run CREATE or SET outside a change — they will silently not persist.

## Creating Nodes and Edges

**Single query (preferred):** Create nodes and edges together — no intermediate COMMIT needed:

```python
client.query("CREATE (:Person {name: 'Jane'})-[:KNOWS]->(:Person {name: 'John'})")
client.query("CHANGE SUBMIT")
```

**Separate queries:** If you create nodes first and edges second, you must COMMIT after the nodes before creating edges:

```python
client.query("CREATE (:Person {name: 'Alice'}), (:Person {name: 'Bob'})")
client.query("COMMIT")   # persist nodes so they're visible for the next query
client.query("MATCH (a:Person {name: 'Alice'}), (b:Person {name: 'Bob'}) CREATE (a)-[:KNOWS]->(b)")
client.query("CHANGE SUBMIT")
```

## CREATE Syntax

```cypher
CREATE (:Person {name: 'Alice', age: 30})
CREATE (:Person {name: 'Mick'})-[:FRIEND_OF]->(:Person {name: 'John'})

-- Multiple nodes at once:
CREATE (:Person {name: 'Alice'}), (:Person {name: 'Bob'}), (:City {name: 'London'})

-- Connect two existing nodes:
MATCH (n:Person {name: 'Alice'}), (m:Person {name: 'Bob'})
CREATE (n)-[:KNOWS]->(m)

-- Create edge and return properties:
MATCH (n:Person {name: 'Alice'}), (m:Person {name: 'Bob'})
CREATE (n)-[:KNOWS]->(m)
RETURN n.name, m.name
```

Rules:
- Every node and edge requires at least one label
- Pure CREATE has no RETURN clause — only add RETURN when combining with MATCH

## SET Syntax (Update Properties)

SET always follows a MATCH. It creates the property if it doesn't already exist.

```cypher
MATCH (n:Person {name: 'Alice'}) SET n.age = 31

-- Multiple properties at once:
MATCH (n:Person {name: 'Alice'}) SET n.score = n.score * 1.1, n.updated = true

-- Expression-based update:
MATCH (p:Product) SET p.discountPrice = p.price * (1 - 0.15)

-- Update an edge property:
MATCH (n:Person {name: 'Alice'})-[e:KNOWS]->(m) SET e.since = 2020

-- Conditional update:
MATCH (n:Person) WHERE n.age < 18 SET n.isMinor = true

-- Set a vector embedding:
MATCH (n:Person {name: 'Alice'}) SET n.emb = [1.2, 2.0, 0.0, 12.0]
```

## Engine Commands for Change Management

These can be issued via `client.query()` or from the CLI:

| Command | Description |
|---------|-------------|
| `CHANGE NEW` | Create a new isolated change |
| `CHANGE SUBMIT` | Merge current change into main |
| `CHANGE DELETE` | Discard current change |
| `CHANGE LIST` | List active uncommitted changes |
| `COMMIT` | Persist intermediate state within a change (needed between separate node/edge creates) |

## Gotchas

- CREATE/SET outside a change do not persist — always use the change workflow
- Creating nodes and edges in separate queries requires `COMMIT` between the two steps
- Pure CREATE has no RETURN clause
- Every node and edge must have at least one label
