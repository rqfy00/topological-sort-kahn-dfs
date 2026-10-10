# topological sort: Kahn's Algorithm vs DFS, Cycle Detection, and the DAG Problems Interviewers Actually Ask

People end up here for three different reasons.

Someone has a homework set due and needs to know why their graph has no valid ordering. Someone is staring at Course Schedule on a screen and can't turn "wait, courses depend on each other" into working code. And someone — usually a backend or build-tooling engineer — has a circular dependency error in a pipeline and wants to know why the tool can't just pick an order.

All three are the same problem. Topological sort takes a directed graph and returns an ordering where every node shows up before the nodes it points to. Here's what actually matters: how the two standard algorithms work, which one to write under interview pressure, how cycles break everything, and where the practical gotchas hide.

## What a topological order actually is

A graph is directed when every edge has a direction. For a node `v`, the in-edges point into it, the out-edges point away from it. The number of in-edges is the in-degree.

A topological order of a directed graph is an ordering of all nodes such that for every edge `(u, v)`, `u` appears before `v`. That's the whole definition, and it has one immediate consequence: the first node in the ordering must have in-degree 0, and the last must have out-degree 0.

Two things trip people up early.

**The ordering is not unique.** For a graph where node 2 depends on both 4 and 5, but 4 and 5 are independent of each other, both `[4, 5, 2]` and `[5, 4, 2]` are valid answers. Same with tasks: if task 3 has no relationship to tasks 2 and 4, it can sit anywhere as long as it lands before task 1, which depends on it. `[4, 2, 3, 1]`, `[4, 3, 2, 1]` and `[3, 4, 2, 1]` are all correct.

**A graph with a cycle has no topological order at all.** This isn't a limitation of the algorithms — it's impossible by definition. If 2 must come before 5, 5 before 4, and 4 before 2, there's no first element you can pick. Any answer you produce is wrong, which is why cycle detection is part of the algorithm and not a separate step you add later.

Graphs where an ordering exists are called DAGs — directed acyclic graphs. That name shows up constantly in build systems, schedulers and data pipelines, and it's the same concept.

## Kahn's algorithm: peel off the zero in-degree nodes

Kahn's approach is a BFS with one modification: only nodes with in-degree 0 are allowed into the queue.

1. Count the in-degree of every node.
2. Push every node with in-degree 0 into a queue.
3. Pop a node, append it to the output order.
4. For each neighbor the node points to, decrement that neighbor's in-degree by 1.
5. If a neighbor's in-degree drops to 0, push it.
6. Repeat until the queue is empty.

python
from collections import deque

def topological_sort(n, edges):
    graph = [[] for _ in range(n)]
    indegree = [0] * n
    for u, v in edges:          # edge u -> v
        graph[u].append(v)
        indegree[v] += 1

    q = deque(i for i in range(n) if indegree[i] == 0)
    order = []
    while q:
        u = q.popleft()
        order.append(u)
        for v in graph[u]:
            indegree[v] -= 1
            if indegree[v] == 0:
                q.append(v)

    return order if len(order) == n else []   # empty list = cycle


That last line is the entire cycle check. If the graph has a cycle, the nodes in the cycle never reach in-degree 0, the queue drains early, and your output is shorter than the node count. You never write a separate DFS-with-colors routine to detect the cycle — the shortfall tells you.

Why it works: at any moment, a node with in-degree 0 has no unfinished prerequisites, so putting it next in the order can't violate any constraint. Removing it (conceptually) is safe, and the decrement step updates exactly the nodes whose last prerequisite just disappeared.

## The DFS version: reverse the finishing times

The other classic algorithm is top-down. Instead of asking "which node can I do now?", it asks "what does this node depend on?" and recurses.

python
def topological_sort_dfs(n, edges):
    graph = [[] for _ in range(n)]
    for u, v in edges:
        graph[u].append(v)

    state = [0] * n     # 0 = unvisited, 1 = in progress, 2 = done
    order = []

    def dfs(u):
        state[u] = 1
        for v in graph[u]:
            if state[v] == 1:
                return False        # back edge -> cycle
            if state[v] == 0 and not dfs(v):
                return False
        state[u] = 2
        order.append(u)             # finished
        return True

    for i in range(n):
        if state[i] == 0 and not dfs(i):
            return []
    order.reverse()                 # reverse postorder
    return order


The key idea: a node is appended only after all of its out-neighbors have finished. So the order is built from the "sinks" backward, and reversing it gives a valid topological order. This is the reverse of DFS postorder, which is the framing used in most algorithms textbooks.

The three-state array matters more than it looks. Marking a node `1` when you enter it and `2` when you leave it lets you detect a *back edge* — an edge pointing to a node that's still on the current recursion stack. A back edge means a cycle. If you only keep a boolean `visited`, you'll happily recurse into a cycle and either loop or return a wrong answer.

One practical warning: this version is recursive, so on a graph with a few hundred thousand nodes you'll blow the Python recursion limit. Kahn's version uses an explicit queue and has no such problem. For Course Schedule and similar interview problems, either is fine — node counts are small.

## Both are O(V + E)

The cost breakdown is honest and unglamorous. Counting in-degrees means walking every edge once. Each node enters and leaves the queue exactly once. Each edge is relaxed once when its source node is popped. Add it up: O(V + E) time, O(V + E) space for the adjacency list.

The naive version you might write first — scan the whole graph for a zero in-degree node, remove it, rescan — is O(V² + E) because you re-scan all the nodes on every iteration. Collecting the zero in-degree nodes once, as updates happen, is what gets you to linear time.

## Which one to write in an interview

Both are correct and both hit the same complexity. The differences are practical.

|  | Kahn's algorithm (BFS) | DFS reverse postorder |
| --- | --- | --- |
| Approach | Iterative, queue-driven | Recursive |
| Cycle detection | Output shorter than node count | Back edge to an in-progress node |
| Needs in-degree array | Yes | No |
| Recursion depth risk | None | Yes, on large graphs |
| Extra problems it unlocks | Lexicographically smallest order (swap queue for min-heap), minimum height trees, task scheduling variants | All paths in a DAG, detecting cycles anywhere in a directed graph |
| Where it feels natural | "What can I do right now?" dependency questions | "What does this thing require?" questions |

Where Kahn's tends to win in interviews: `Course Schedule`, `Course Schedule II`, `Alien Dictionary`-shaped problems, and anything where you also need to reason about batches or levels. Where DFS fits better: problems that are already recursive, or when you want a single traversal that can also answer "is there a cycle?" without extra bookkeeping.

A reasonable def≥ault when you're nervous: write Kahn's. There's no recursion limit, the state is a flat array you can print while debugging, and explaining the in-degree decrement out loud makes the interviewer confident you understand why it works and not just what to type.

## When "any valid order" isn't good enough

Interview problems often stop at *a* valid order. Real systems and harder variants don't.

**Unique ordering.** A DAG has exactly one topological order only when the graph contains a path through every node, with extra edges allowed to skip forward along it. Kahn's gives you a cheap test: if the queue ever holds more than one node when you pop, two orderings exist, so the answer isn't unique.

**Lexicographically smallest ordering.** Swap the queue for a min-heap. Instead of any zero in-degree node, always take the smallest one. That's the whole change, and it's a common follow-up question.

**All orderings.** Enumerate by choosing any zero in-degree node at each step and recursing. The number of valid orderings can be exponential, so this is a different class of problem — worth recognizing rather than attempting to optimize.

**Counting orderings.** That's DP over subsets or over the DAG structure, and it's usually the "hard" tier of the same problem family.

## Where this shows up outside of interviews

Topological sort isn't an academic exercise. You've been relying on it without noticing:

- **Package installs.** `pip`, `npm` and `apt` compute an install order from a dependency graph. The "circular dependency" error is the algorithm reporting that no topological order exists, and the package manager is telling you to break the cycle by hand.
- **Build systems.** Make, Bazel and most task runners topologically sort targets so dependencies compile first. Parallel execution usually works level by level through the ordering.
- **Course prerequisites.** The problem is stated about courses precisely because the analogy is exact: a prerequisite edge from course B to course A, and you need an order that respects every edge.
- **Spreadsheet and formula evaluation.** Cells that reference other cells form a DAG, and recalculation follows a topological order so each formula reads already-computed values.
- **DP on DAGs.** If a problem's state transitions aren't already in a natural order — longest path in a DAG, for example — topologically sorting the states gives you a valid processing order for free. This is where a lot of people first notice that topological sort and dynamic programming are the same idea wearing different clothes.

If you're getting a cycle error and you're sure your data has no loop, the bug is usually in how you built the edges — typically reversed direction, which turns a valid DAG into a self-contradicting one.

## Mistakes that cost you the interview

- **Decrementing the wrong side.** The edge points from the prerequisite to the dependent node. Decrement the in-degree of the node being pointed *to*, not the one you just popped.
- **Seeding the queue after the loop instead of before.** Every zero in-degree node goes in at the start, not just the one you find first.
- **Using `visited` where `in-degree` is required.** In Kahn's, a node can be visited-looking and still have unsatisfied prerequisites. The in-degree array is the source of truth.
- **Forgetting the length check.** Without `len(order) == n`, a cyclic graph silently produces a truncated answer instead of reporting failure. In `Course Schedule` this is the difference between accepted and wrong answer.
- **Boolean visited in the DFS version.** You'll miss back edges and fail to detect cycles.
- **Assuming the first valid order you compute is the answer.** If the problem asks for the lexicographically smallest order, plain BFS won't give it.

## Practicing on the graph section instead of grinding randomly

Most people who get stuck on `Course Schedule` aren't stuck on code. They're stuck on not recognizing that "tasks with prerequisites" is a graph problem at all. That recognition is a skill you train, not a fact you memorize.

AlgoMonster's [👉 graph and topological sort lesson](https://algo.monster?fpr=li69) takes a specific approach: it starts from the vocabulary (in-edge, out-edge, in-degree, out-degree, dependency), then shows why a cyclic graph has no ordering, then works through Kahn's algorithm step by step with diagrams before showing implementations in Python, Java, JavaScript and C++. There's a dedicated section comparing topological sort to plain BFS, which is exactly the distinction that confuses people who half-remember BFS.

Then it hands you the problems in order: `Course Schedule` (can you finish all courses — the cycle check), `Course Schedule II` (return the actual ordering), and the harder variants. Code, editorials and complexity notes for each. The material is text and diagrams rather than video, which is either a selling point or a dealbreaker depending on how you learn — there's no video walkthrough library, so if you need someone talking through the code, this isn't that.

Two things worth knowing about the platform's shape. First, it's pattern-organized rather than problem-count-organized; you're meant to learn a template and recognize where it applies, not collect solved problems. Second, it's built by engineers with competitive programming backgrounds, and it currently includes bonus courses on CS foundations and building an LLM from scratch.

## Plans and pricing

The official subscribe page shows three options, and the paid prices shift with sales — they've been observed at materially different levels on the same page at different times, so treat the numbers below as the range currently displayed rather than a fixed contract.

| Plan | What you get | Price shown | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited sample access to a subset of lessons and problems | $0 | — | [ Create a free account and try the graph lessons](https://algo.monster?fpr=li69) |
| Pro – Yearly | Unlock all patterns, lessons and problems, unlimited AI assistant, unlimited Speedrun, monthly-updated real interview questions, 12 months of access | Listed at $119/year (~$9.92/mo); sale pricing has shown $69.99/year (~$5.84/mo) | Annual, cancel or upgrade anytime | [ Check the current yearly price](https://algo.monster?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan plus lifetime updates, no renewal | Listed at $459 once; a promotional rate of $89 has appeared on the same page | One-time payment | [ See the lifetime plan details](https://algo.monster?fpr=li69) |

Two caveats that matter more than the specific numbers. There's no monthly option — it's annual or lifetime, so there's no way to pay for just the six weeks you need for one interview loop. And the free tier is genuinely limited: it's enough to see whether the text-and-diagram teaching style clicks, not enough to prepare for anything.

The math you should actually run: if your loop is in two months, the annual plan is the obvious choice and the lifetime plan is money you won't use again for a year or two. If you interview repeatedly — every promotion cycle, every job hop — lifetime breaks even against a couple of annual renewals. Nobody can tell you which case you are.

## Questions that come up a lot

**Do I need a DAG for topological sort?** Yes. If the graph has a cycle, no valid ordering exists, and the correct output is an error or empty result. This is the one thing to say out loud when an interviewer asks how you'd handle the impossible case.

**Is topological sort the same as DFS?** No. DFS is a traversal; topological sort is an ordering. One common topological sort is *built on* DFS — you run DFS and reverse the postorder. The DFS traversal order itself is not a topological order.

**Can there be several correct answers?** Usually yes. Any independent nodes can be swapped freely. Only when the graph has a single path covering all nodes is the ordering unique.

**Which is faster, Kahn's or DFS?** Same asymptotic O(V + E). The practical difference is recursion depth and how much extra state you're carrying. On interview-sized inputs, pick whichever you can write correctly under time pressure.

## The short version

Write Kahn's unless you have a specific reason not to: count in-degrees, queue the zeros, decrement as you pop, and check that the output length matches the node count. That last check is your cycle detection, and forgetting it is the single most common way people fail `Course Schedule`.

If the pattern still doesn't click, the gap is usually recognition rather than implementation — and that's trainable. [👉 Work through the topological sort lesson and its problem set](https://algo.monster?fpr=li69) with the decision framework alongside it, then come back to a fresh Course Schedule variant and see whether you identify the approach in under a minute.
