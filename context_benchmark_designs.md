# Context Utilization Benchmark Designs

## Purpose
Test the hypothesis that structured, schema-aligned input allows transformers to utilize more of their context window effectively than equivalent natural language prose.

## General Parameters (All Tests)

```python
SCALE_PARAMS = {
    "100k": {"multiplier": 0.05},
    "500k": {"multiplier": 0.25},
    "1M":   {"multiplier": 0.5},
    "2M":   {"multiplier": 1.0},
}
```

---

# Benchmark 1: The Ledger Problem

## Concept
Track inventory across warehouses through a long sequence of transactions. Every transaction matters because the final answer depends on the cumulative state.

## Scaling Parameters

```python
LEDGER_PARAMS = {
    "num_warehouses": 100,      # W
    "num_skus": 50,             # S
    "transactions_per_wh": 400, # T
    # Total records ≈ W × T = 40,000 base
    # Scale by multiplier for target context
}
```

## Data Generation Logic

```python
import random
import json

def generate_ledger(params, seed=42):
    random.seed(seed)

    warehouses = [f"WH-{i:04d}" for i in range(params["num_warehouses"])]
    skus = [f"SKU-{i:04d}" for i in range(params["num_skus"])]

    # Initialize inventory (random 10-100 per SKU per warehouse)
    inventory = {
        wh: {sku: random.randint(10, 100) for sku in skus}
        for wh in warehouses
    }

    transactions = []
    timestamp = 1704067200  # Jan 1, 2024

    for _ in range(params["num_warehouses"] * params["transactions_per_wh"]):
        action = random.choice(["sale", "restock", "transfer", "adjustment"])
        wh = random.choice(warehouses)
        sku = random.choice(skus)

        if action == "sale":
            qty = random.randint(1, min(10, inventory[wh][sku]))
            inventory[wh][sku] -= qty
            transactions.append({
                "ts": timestamp,
                "warehouse": wh,
                "sku": sku,
                "action": "sale",
                "qty": -qty
            })

        elif action == "restock":
            qty = random.randint(5, 50)
            inventory[wh][sku] += qty
            transactions.append({
                "ts": timestamp,
                "warehouse": wh,
                "sku": sku,
                "action": "restock",
                "qty": qty
            })

        elif action == "transfer":
            other_wh = random.choice([w for w in warehouses if w != wh])
            qty = random.randint(1, min(20, inventory[wh][sku]))
            inventory[wh][sku] -= qty
            inventory[other_wh][sku] += qty
            transactions.append({
                "ts": timestamp,
                "warehouse": wh,
                "sku": sku,
                "action": "transfer_out",
                "qty": -qty,
                "dest": other_wh
            })
            transactions.append({
                "ts": timestamp,
                "warehouse": other_wh,
                "sku": sku,
                "action": "transfer_in",
                "qty": qty,
                "source": wh
            })

        elif action == "adjustment":
            adj = random.randint(-5, 5)
            inventory[wh][sku] = max(0, inventory[wh][sku] + adj)
            transactions.append({
                "ts": timestamp,
                "warehouse": wh,
                "sku": sku,
                "action": "adjustment",
                "qty": adj
            })

        timestamp += random.randint(60, 3600)

    # Pick a random target for the question
    target_wh = random.choice(warehouses)
    target_sku = random.choice(skus)
    answer = inventory[target_wh][target_sku]

    return {
        "initial_inventory": {wh: dict(inv) for wh, inv in inventory.items()},  # snapshot before transactions
        "transactions": transactions,
        "question": f"What is the current inventory of {target_sku} at {target_wh}?",
        "answer": answer
    }
```

## Output Formats

### Structured (JSON)
```json
{"ts":1704067200,"warehouse":"WH-0042","sku":"SKU-0017","action":"sale","qty":-3}
{"ts":1704067845,"warehouse":"WH-0089","sku":"SKU-0017","action":"transfer_out","qty":-12,"dest":"WH-0042"}
```

### Prose
```
At 12:00:00 on January 1st, Warehouse 42 sold 3 units of SKU-0017.
At 12:10:45 on January 1st, Warehouse 89 transferred 12 units of SKU-0017 to Warehouse 42.
```

## Verification
Sum all qty changes for target (warehouse, sku) pair + initial inventory = answer.

---

# Benchmark 2: Distributed Constraint Satisfaction

## Concept
N entities each have M attributes. Constraints are scattered throughout the document linking entity attributes. Only one valid assignment exists.

## Scaling Parameters

```python
CONSTRAINT_PARAMS = {
    "num_entities": 200,        # E
    "num_attributes": 10,       # A
    "attribute_domains": 8,     # D (possible values per attribute)
    "num_constraints": 2000,    # C
    # Total records ≈ E×A + C = 4,000 base
}
```

## Data Generation Logic

```python
def generate_constraints(params, seed=42):
    random.seed(seed)

    entities = [f"E-{i:04d}" for i in range(params["num_entities"])]
    attributes = [f"attr_{i}" for i in range(params["num_attributes"])]
    domains = {
        "attr_0": ["red", "blue", "green", "yellow", "orange", "purple", "black", "white"],
        "attr_1": ["small", "medium", "large", "xlarge", "tiny", "huge", "micro", "mega"],
        "attr_2": list(range(1, 9)),
        # ... etc for each attribute
    }

    # Generate a valid assignment first (ground truth)
    assignment = {
        entity: {attr: random.choice(domains[attr]) for attr in attributes}
        for entity in entities
    }

    # Generate constraints that are satisfied by this assignment
    constraints = []

    for _ in range(params["num_constraints"]):
        constraint_type = random.choice([
            "equality",      # E1.attr_a == E2.attr_a
            "inequality",    # E1.attr_a != value
            "implication",   # IF E1.attr_a == X THEN E2.attr_b == Y
            "mutual",        # E1.attr_a == E2.attr_b (cross-attribute)
        ])

        e1 = random.choice(entities)
        attr1 = random.choice(attributes)

        if constraint_type == "equality":
            e2 = random.choice([e for e in entities if e != e1])
            # Make it true for our assignment
            if assignment[e1][attr1] == assignment[e2][attr1]:
                constraints.append({
                    "type": "eq",
                    "e1": e1, "a1": attr1,
                    "e2": e2, "a2": attr1
                })

        elif constraint_type == "inequality":
            wrong_values = [v for v in domains[attr1] if v != assignment[e1][attr1]]
            if wrong_values:
                constraints.append({
                    "type": "neq",
                    "entity": e1,
                    "attr": attr1,
                    "value": random.choice(wrong_values)
                })

        elif constraint_type == "implication":
            e2 = random.choice(entities)
            attr2 = random.choice(attributes)
            constraints.append({
                "type": "impl",
                "if_entity": e1, "if_attr": attr1, "if_value": assignment[e1][attr1],
                "then_entity": e2, "then_attr": attr2, "then_value": assignment[e2][attr2]
            })

        elif constraint_type == "mutual":
            e2 = random.choice([e for e in entities if e != e1])
            attr2 = random.choice(attributes)
            if assignment[e1][attr1] == assignment[e2][attr2]:
                constraints.append({
                    "type": "mut",
                    "e1": e1, "a1": attr1,
                    "e2": e2, "a2": attr2
                })

    # Shuffle constraints to distribute information
    random.shuffle(constraints)

    # Pick target
    target_entity = random.choice(entities)
    target_attr = random.choice(attributes)
    answer = assignment[target_entity][target_attr]

    return {
        "entities": entities,
        "attributes": attributes,
        "domains": domains,
        "constraints": constraints,
        "question": f"What is the {target_attr} of {target_entity}?",
        "answer": answer,
        "_ground_truth": assignment  # For verification only
    }
```

## Output Formats

### Structured (JSON)
```json
{"type":"eq","e1":"E-0042","a1":"attr_0","e2":"E-0189","a2":"attr_0"}
{"type":"neq","entity":"E-0042","attr":"attr_0","value":"purple"}
{"type":"impl","if_entity":"E-0042","if_attr":"attr_0","if_value":"blue","then_entity":"E-0100","then_attr":"attr_2","then_value":7}
```

### Prose
```
Entity 42 and Entity 189 have the same color.
Entity 42 is not purple.
If Entity 42 is blue, then Entity 100 has a size rating of 7.
```

## Verification
Run constraint solver on extracted constraints; should yield unique solution matching answer.

---

# Benchmark 3: The Network Flow Problem

## Concept
A directed graph with weighted edges. Weights change over time (events). Question asks for optimal path or total flow at a specific time.

## Scaling Parameters

```python
NETWORK_PARAMS = {
    "num_nodes": 500,           # N
    "num_edges": 2000,          # E (sparse graph)
    "num_weight_changes": 8000, # W (events that modify edge weights)
    # Total records ≈ N + E + W = 10,500 base
}
```

## Data Generation Logic

```python
def generate_network(params, seed=42):
    random.seed(seed)

    nodes = [f"N-{i:04d}" for i in range(params["num_nodes"])]

    # Create base graph (ensure connected)
    edges = {}

    # First, create a spanning tree for connectivity
    shuffled = nodes.copy()
    random.shuffle(shuffled)
    for i in range(1, len(shuffled)):
        parent = shuffled[random.randint(0, i-1)]
        child = shuffled[i]
        edges[(parent, child)] = random.randint(1, 100)

    # Add additional random edges
    while len(edges) < params["num_edges"]:
        n1 = random.choice(nodes)
        n2 = random.choice([n for n in nodes if n != n1])
        if (n1, n2) not in edges:
            edges[(n1, n2)] = random.randint(1, 100)

    # Generate weight change events
    events = []
    timestamp = 1704067200

    for _ in range(params["num_weight_changes"]):
        edge = random.choice(list(edges.keys()))
        change_type = random.choice(["set", "add", "multiply"])

        if change_type == "set":
            new_weight = random.randint(1, 100)
            edges[edge] = new_weight
            events.append({
                "ts": timestamp,
                "edge": edge,
                "action": "set",
                "value": new_weight
            })
        elif change_type == "add":
            delta = random.randint(-20, 50)
            edges[edge] = max(1, edges[edge] + delta)
            events.append({
                "ts": timestamp,
                "edge": edge,
                "action": "add",
                "value": delta
            })
        elif change_type == "multiply":
            factor = random.choice([0.5, 0.75, 1.25, 1.5, 2.0])
            edges[edge] = max(1, int(edges[edge] * factor))
            events.append({
                "ts": timestamp,
                "edge": edge,
                "action": "multiply",
                "value": factor
            })

        timestamp += random.randint(60, 600)

    # Compute shortest path for question
    # (Use Dijkstra on final state)
    source = random.choice(nodes)
    dest = random.choice([n for n in nodes if n != source])

    # Simplified: compute answer via Dijkstra
    answer = dijkstra_shortest_path(nodes, edges, source, dest)

    return {
        "nodes": nodes,
        "initial_edges": [(e[0], e[1], w) for e, w in edges.items()],
        "events": events,
        "question": f"What is the shortest path distance from {source} to {dest} after all events?",
        "answer": answer
    }

def dijkstra_shortest_path(nodes, edges, source, dest):
    import heapq
    dist = {n: float('inf') for n in nodes}
    dist[source] = 0
    pq = [(0, source)]

    adj = {n: [] for n in nodes}
    for (u, v), w in edges.items():
        adj[u].append((v, w))

    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]:
            continue
        for v, w in adj[u]:
            if dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
                heapq.heappush(pq, (dist[v], v))

    return dist[dest] if dist[dest] != float('inf') else -1
```

## Output Formats

### Structured (JSON)
```json
{"type":"node","id":"N-0042"}
{"type":"edge","from":"N-0042","to":"N-0189","weight":47}
{"type":"event","ts":1704067200,"from":"N-0042","to":"N-0189","action":"set","value":23}
```

### Prose
```
Node 42 exists in the network.
There is a connection from Node 42 to Node 189 with initial weight 47.
At 12:00:00 on January 1st, the connection from Node 42 to Node 189 was updated to weight 23.
```

## Verification
Replay all events on graph, run Dijkstra, compare to answer.

---

# Test Harness Design

```python
class ContextBenchmark:
    def __init__(self, benchmark_type, scale="1M", seed=42):
        self.type = benchmark_type
        self.scale = scale
        self.seed = seed
        self.data = None
        self.structured_doc = None
        self.prose_doc = None

    def generate(self):
        """Generate the benchmark data."""
        if self.type == "ledger":
            self.data = generate_ledger(scale_params(LEDGER_PARAMS, self.scale), self.seed)
        elif self.type == "constraint":
            self.data = generate_constraints(scale_params(CONSTRAINT_PARAMS, self.scale), self.seed)
        elif self.type == "network":
            self.data = generate_network(scale_params(NETWORK_PARAMS, self.scale), self.seed)

    def to_structured(self):
        """Convert to JSON-lines format."""
        # Implementation per benchmark type
        pass

    def to_prose(self):
        """Convert to natural language prose."""
        # Implementation per benchmark type
        pass

    def verify_answer(self, response):
        """Check if model response matches ground truth."""
        return extract_number(response) == self.data["answer"]

    def token_count(self, format="structured"):
        """Estimate token count for given format."""
        doc = self.structured_doc if format == "structured" else self.prose_doc
        return len(doc) // 4  # Rough estimate

def scale_params(base_params, scale):
    """Apply scale multiplier to parameters."""
    multiplier = SCALE_PARAMS[scale]["multiplier"]
    return {k: int(v * multiplier) if isinstance(v, int) else v
            for k, v in base_params.items()}

def run_benchmark(model_api, benchmark, formats=["structured", "prose"]):
    """Run benchmark and collect results."""
    results = {}

    for fmt in formats:
        doc = benchmark.to_structured() if fmt == "structured" else benchmark.to_prose()
        prompt = f"{doc}\n\nQuestion: {benchmark.data['question']}\nAnswer with just the number:"

        response = model_api.complete(prompt)
        correct = benchmark.verify_answer(response)

        results[fmt] = {
            "correct": correct,
            "response": response,
            "expected": benchmark.data["answer"],
            "token_count": benchmark.token_count(fmt)
        }

    return results
```

---

# Experimental Protocol

1. **Generate benchmarks** at each scale (100k, 500k, 1M, 2M) with 10 different seeds
2. **Run each benchmark** in both formats (structured, prose)
3. **Collect metrics**:
   - Accuracy (binary)
   - Response correctness margin (how far off)
   - Token counts for each format
4. **Statistical analysis**:
   - Paired t-test structured vs prose at each scale
   - Accuracy degradation curve as scale increases
5. **Ablation studies**:
   - Shuffle order of records (test position bias)
   - Corrupt X% of records (test robustness)
   - Vary schema verbosity (terse keys vs descriptive)

---

# Expected Outcomes

**If hypothesis is correct:**
- Structured format maintains accuracy at larger scales
- Prose degrades faster as context grows
- Divergence point reveals "effective context limit" for unstructured input

**Alternative findings:**
- No difference (models handle both equally)
- Prose better (training distribution favors natural language)
- Task-dependent (some benchmarks favor structure, others don't)

---

# Implementation Notes

- Use tiktoken for accurate token counting
- Test on: GPT-4-turbo (128k), Claude (200k), Gemini 1.5 (1M+)
- Log full responses for qualitative analysis
- Consider chain-of-thought prompting as additional variable
