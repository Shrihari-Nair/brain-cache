# Dijkstra's Algorithm — Explained via Network Delay Time (LeetCode 743)

## The Problem

You're given a network of `n` nodes labeled `1` to `n`, and a list of directed, weighted edges `times`, where each `times[i] = (u, v, w)` means it takes `w` time units for a signal to travel from node `u` to node `v`.

A signal is sent from node `k`. Return the **minimum time** for the signal to reach **all** `n` nodes. If it's impossible for all nodes to receive the signal, return `-1`.

This is a classic **single-source shortest path** problem — and Dijkstra's algorithm is the standard tool for it, provided all edge weights are non-negative (which they are here).

---

## The Core Idea Behind Dijkstra's Algorithm

Dijkstra's algorithm finds the shortest distance from a single starting node to every other reachable node in a weighted graph (with non-negative weights). The core idea is:

> **Always expand outward from the node you can currently reach with the smallest known total cost.**

This is a **greedy** strategy: once you've found the shortest path to a node (by always processing the cheapest option first), you can trust that distance is final and never needs to be revisited — because any other path to that node would have to go through a node with an even higher or equal cost, and since weights are non-negative, that path can never be shorter.

The trick to implementing this efficiently is using a **min-heap (priority queue)** so that at every step, you always pull out the currently-cheapest reachable node first.

---

## Walking Through the Code

```python
class Solution:
    def networkDelayTime(self, times: List[List[int]], n: int, k: int) -> int:
        edges = defaultdict(list)
        for u, v, w in times:
            edges[u].append((v, w))
        heap = [(0, k)]
        t = 0
        visit = set()
        while heap:
            w1, n1 = heapq.heappop(heap)
            if n1 in visit:
                continue
            visit.add(n1)
            t = max(t, w1)
            for n2, w2 in edges[n1]:
                if n2 not in visit:
                    heapq.heappush(heap, (w1 + w2, n2))
        return t if len(visit) == n else -1
```

### Step 1 — Build the adjacency list

```python
edges = defaultdict(list)
for u, v, w in times:
    edges[u].append((v, w))
```

This converts the flat list of edges into a graph representation: for each node `u`, `edges[u]` holds a list of `(neighbor, weight)` pairs — i.e., "from `u`, you can reach `v` at cost `w`." This lets you quickly look up all outgoing edges from any node during traversal, in O(1) average time per lookup.

### Step 2 — Initialize the priority queue (min-heap)

```python
heap = [(0, k)]
```

This is the heart of Dijkstra's algorithm. The heap stores `(distance_so_far, node)` pairs, and because Python's `heapq` is a **min-heap**, it always pops the pair with the **smallest first element** — i.e., the smallest accumulated distance.

We start by pushing `(0, k)`: the source node `k` is reachable from itself at cost `0`.

**Why a heap instead of just a queue or a list?** Because at every step, we need the cheapest unvisited option available *so far*, not just the next one in insertion order. A heap gives us O(log n) insertion and O(log n) extraction of the minimum, which is what makes Dijkstra's efficient (O((V + E) log V) overall) rather than needing to linearly scan for the minimum every time (O(V²), which is the naive alternative).

### Step 3 — The main loop: greedily expand the cheapest frontier

```python
while heap:
    w1, n1 = heapq.heappop(heap)
    if n1 in visit:
        continue
    visit.add(n1)
    t = max(t, w1)
```

Each iteration:
1. **Pop the cheapest `(distance, node)` pair** from the heap. Because of how min-heaps work, this is guaranteed to be the smallest distance currently known across the entire frontier of "reachable but maybe not yet finalized" nodes.
2. **Skip if already visited.** A node can be pushed onto the heap multiple times (once for each different path that reaches it), but only the **first time we pop it** (which is guaranteed to be via its shortest path, thanks to the greedy min-heap ordering) do we treat it as "finalized." Any later, more expensive entries for the same node are just stale and get ignored.
3. **Mark it visited and finalize its distance.** Once popped for the first time, `w1` *is* the shortest distance to `n1` — this can be trusted permanently because of the non-negative-weight guarantee (explained further below).
4. **Track the running answer `t`.** Since we want the time for the signal to reach *all* nodes, we need the **maximum** of all the individual shortest distances — the last node to receive the signal determines the total delay.

### Step 4 — Relax the neighbors (push new candidate distances)

```python
for n2, w2 in edges[n1]:
    if n2 not in visit:
        heapq.heappush(heap, (w1 + w2, n2))
```

For every neighbor `n2` of the just-finalized node `n1`, compute the total cost to reach `n2` via `n1`: `w1 + w2`. Push this as a new candidate distance for `n2` onto the heap.

Note: we don't check "is this better than a previously pushed distance for `n2`?" before pushing — we just push it, and let the heap's ordering plus the `if n1 in visit: continue` check at pop-time sort out which pushed distance actually "wins" (the smallest one will always be popped first). This is a common simplification of Dijkstra's called the **"lazy deletion"** approach — instead of maintaining a decrease-key operation on the heap (which `heapq` doesn't support natively), we just allow duplicate/stale entries to exist in the heap and filter them out lazily when popped.

### Step 5 — Final check

```python
return t if len(visit) == n else -1
```

If we finalized distances for all `n` nodes, `t` (the maximum shortest-distance found) is the answer — this is the time for the signal to reach the *farthest* node, which is when "all nodes have received the signal."

If `len(visit) < n`, it means some nodes were never reachable from `k` at all — the heap ran empty before we could visit everyone, so we return `-1`.

---

## Why the Greedy Approach Is Provably Correct (Non-Negative Weights)

This is the part that makes Dijkstra's *not* just a heuristic, but a provably correct algorithm — and it hinges entirely on the fact that **all edge weights are non-negative**.

**Claim:** The first time a node is popped from the heap, the distance associated with it is guaranteed to be its true shortest distance from the source.

**Why:** Suppose node `X` is popped with distance `d`. Because it's a min-heap, `d` is the smallest distance value currently present anywhere in the heap. Now suppose, for contradiction, that there's actually a shorter path to `X` that we haven't discovered yet. That undiscovered path must pass through some other node `Y` that hasn't been finalized yet (otherwise we would already have discovered it). But the distance to reach `Y` alone is already `≥ d` (since `d` is the smallest value in the heap, and `Y`'s best known distance, if better, would have been popped before `X`). Since edge weights are **non-negative**, continuing from `Y` to `X` can only add more distance, never subtract. So any path through `Y` would end up being `≥ d` as well — contradicting the assumption that a shorter path exists.

This is precisely why Dijkstra's algorithm **fails** on graphs with negative edge weights — the "once popped, distance is final" guarantee breaks down, because a later, cheaper path could theoretically "catch up" and undercut an already-finalized distance. (For graphs with negative weights, you'd need Bellman-Ford instead.)

---

## Complexity Analysis

- **Time:** `O((V + E) log V)`, where `V` is the number of nodes and `E` is the number of edges.
  - Each edge can cause at most one heap push → `O(E log V)` for all pushes/pops combined (heap operations are `O(log V)` each, and heap size is bounded by `O(E)`).
  - Building the adjacency list is `O(E)`.
- **Space:** `O(V + E)` — for the adjacency list, the visited set, and the heap (which can hold up to `O(E)` entries in the worst case, due to the lazy-deletion approach allowing duplicates).

---

## Key Takeaways for Applying Dijkstra's Elsewhere

1. **Use a min-heap** to always process the currently-cheapest frontier node next — this is what makes the algorithm greedy and efficient.
2. **Track visited nodes** and skip reprocessing them — this is what prevents infinite loops and redundant work, and is what makes "lazy deletion" of stale heap entries work correctly.
3. **Push new candidate distances freely, without checking for duplicates in the heap** — correctness is preserved because the heap's ordering ensures the true minimum for each node is always popped first; anything else is discarded harmlessly when encountered.
4. **This only works with non-negative edge weights.** If you see negative weights in a problem, Dijkstra's is the wrong tool — reach for Bellman-Ford instead.
5. **The heap always stores `(cost, node)` tuples** (cost first!) — this is essential for Python's `heapq`, since tuples are compared element-by-element, and you want it comparing by cost, not by node ID.