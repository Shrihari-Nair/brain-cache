# Backtracking, Explained via "Unique 3-Digit Even Numbers" (LeetCode 3483)

## The Problem

Given an array of digits `digits` (values `0-9`, possibly with repeats), count how many **unique** 3-digit even numbers can be formed, where:
- Each digit's usage in the number cannot exceed how many times it appears in the input array.
- The number cannot have a leading zero.
- The number must be even.

This problem is a great vehicle for learning backtracking because it combines three separate concerns — **combinatorial generation**, **positional constraints**, and **duplicate avoidance** — and shows how to layer them cleanly.

---

## What Backtracking Actually Is

Backtracking is a systematic way to explore all possible configurations of a solution by **building it incrementally, one decision at a time**, and **undoing (backtracking) a decision** once you've fully explored what it leads to — so you can try the next alternative at that same decision point.

The pattern always has this shape:

```
def backtrack(partial_solution):
    if partial_solution is complete:
        record it / count it
        return
    for each possible next choice:
        if choice is invalid given current state:
            skip it
        make the choice (update state)
        backtrack(partial_solution + choice)
        undo the choice (restore state)   <-- the "backtrack" step
```

The "undo" step is what makes it *backtracking* rather than plain recursion — it's what lets you reuse the same state variables (like `used[]` or `path[]`) across many different branches of exploration, instead of allocating fresh copies at every recursive call.

---

## The Solution

```python
class Solution:
    def totalNumbers(self, digits: List[int]) -> int:
        digits.sort()
        n = len(digits)
        used = [False] * n
        count = 0
        
        def backtrack(path):
            nonlocal count
            if len(path) == 3:
                count += 1
                return
            
            for i in range(n):
                if used[i]:
                    continue
                if i > 0 and digits[i] == digits[i-1] and not used[i-1]:
                    continue
                if len(path) == 0 and digits[i] == 0:
                    continue
                if len(path) == 2 and digits[i] % 2 != 0:
                    continue
                
                used[i] = True
                path.append(digits[i])
                backtrack(path)
                path.pop()
                used[i] = False
        
        backtrack([])
        return count
```

---

## Nuance 1: Track *positions*, not *values*

A very common early mistake in backtracking problems involving duplicates is to track "which values have I used" instead of "which specific array slots have I used."

Here, we use:
```python
used = [False] * n   # one flag per INDEX in the sorted array
```

This matters because the input can have repeated digits — e.g., `digits = [2, 2, 8]`. If you tracked usage by *value* (e.g., a `set()` of used values), you'd only ever be able to use a `2` **once total**, even though the input legitimately contains two `2`s and the resulting number is allowed to use both.

**Rule of thumb:** whenever a backtracking problem's input can contain duplicate *values* but each occurrence is independently available for use, track usage by **index**, not by value.

---

## Nuance 2: The "make a choice → recurse → undo the choice" cycle

```python
used[i] = True
path.append(digits[i])
backtrack(path)
path.pop()
used[i] = False
```

This four-line pattern is the heartbeat of backtracking. Walk through it slowly:

1. **`used[i] = True`** — commit to using this index for the current position.
2. **`path.append(digits[i])`** — commit this digit to the partial solution being built.
3. **`backtrack(path)`** — recurse, exploring everything that can happen *given this choice*.
4. **`path.pop()`** — once that recursive exploration is fully done (all its branches exhausted), remove the digit from `path` — this is the "backtrack."
5. **`used[i] = False`** — free up this index again, so a *sibling* branch (a different choice at this same recursion level) can potentially use it.

**Why undo at all?** Because `path` and `used` are **shared, mutated state** — not copies. If you didn't pop/reset after the recursive call returns, the next iteration of the `for` loop (trying a *different* `i`) would incorrectly see the previous choice still "baked in," corrupting every subsequent branch.

---

## Nuance 3: Positional constraints — check them relative to `len(path)`, not the digit's index

```python
if len(path) == 0 and digits[i] == 0:
    continue      # no leading zero — this is the FIRST digit we're placing
if len(path) == 2 and digits[i] % 2 != 0:
    continue      # last digit must be even — this is the THIRD digit we're placing
```

This is a subtlety worth internalizing: `len(path)` tells you **which position in the final number you're currently deciding** (0 = hundreds place, 1 = tens place, 2 = units place) — completely independent of `i` (which just indexes into the sorted `digits` array to pick a candidate value).

Many backtracking problems have constraints that depend on **where you are in the sequence being built**, not on the raw index of the candidate. Always ask: "is this constraint about the *value*, or about the *position*?" — and check `len(path)` (or an explicit `depth` parameter) for position-based rules.

---

## Nuance 4: Skipping duplicates — the trickiest and most important nuance here

```python
if i > 0 and digits[i] == digits[i-1] and not used[i-1]:
    continue
```

This single line prevents the algorithm from generating the **same final number multiple times** through different, redundant paths. It deserves a careful trace.

**Why duplicates happen without this check:**

Suppose `digits = [2, 2, 3]` (already sorted) and we're choosing the **hundreds digit**. Without any duplicate check, the loop would try:
- `i=0` → pick `digits[0]=2` → recurse
- `i=1` → pick `digits[1]=2` → recurse (again!)
- `i=2` → pick `digits[2]=3` → recurse

Choosing `i=0` and choosing `i=1` both place the **same value** (`2`) into the hundreds slot. Since the two `2`s are indistinguishable in the final number, everything that follows from `i=0` and everything that follows from `i=1` would produce **identical sets of final numbers** — pure duplicated, wasted work.

**How the check prevents this — and why `not used[i-1]` specifically:**

The condition `digits[i] == digits[i-1] and not used[i-1]` reads as: *"I'm equal to my predecessor, and my predecessor is currently just sitting unused."* This exact situation only arises when we're about to pick the **second (or later) copy of a repeated value while an earlier, identical copy remains untouched at this level of recursion** — which is precisely the redundant branch we want to skip.

**Why not simply `digits[i] == digits[i-1]` alone (without the `used[i-1]` check)?** Because that would *incorrectly* also block legitimate cases where you genuinely need to use **both** copies of a repeated digit (e.g., you already placed `digits[0]=2` earlier in the recursion, and are now — validly — trying to place `digits[1]=2` in a later slot). In that scenario, `used[0]` is `True`, so `not used[i-1]` is `False`, and the skip condition doesn't trigger — correctly allowing the second `2` to be used.

**The intuition in one sentence:** *among a run of equal values, only ever pick them in strict left-to-right order at any given branching point* — if you're about to jump to a later duplicate while an earlier identical one is still available, you're re-deriving a branch that's already being explored (or already was explored) via the earlier index.

---

## Nuance 5: Why sorting first is a prerequisite for the duplicate-skip trick

```python
digits.sort()
```

The duplicate-skip check (`digits[i] == digits[i-1]`) only works if equal values are **adjacent** in the array. If the array weren't sorted, two `2`s could be scattered far apart (e.g., `digits = [2, 5, 2]`), and comparing `digits[i]` to `digits[i-1]` would tell you nothing useful about whether they're duplicates of each other.

This is a recurring theme across many backtracking problems that need duplicate avoidance (Permutations II, Subsets II, Combination Sum II) — **sorting is almost always step one** whenever duplicate-skipping logic is involved.

---

## Putting It All Together: Tracing a Small Example

Let's trace `digits = [2, 2, 4]` (already sorted).

**Choosing the hundreds digit (`len(path) == 0`):**
- `i=0`, `digits[0]=2`: not used, no duplicate issue (`i=0` so the check doesn't apply), not zero → valid. **Choose it.**
  - `i=1`, `digits[1]=2`: not used, `digits[1]==digits[0]` but `used[0]` is now `True` → skip condition doesn't trigger → valid. **Choose it.**
    - `i=2`, `digits[2]=4`: not used, no duplicate issue, and `len(path)==2` so check evenness → `4` is even → valid. **Choose it.**
      - `path = [2,2,4]`, length 3 → **count += 1** (number: 224)
    - backtrack, backtrack...
  - Back at `len(path)==1`, try `i=2`: `digits[2]=4` → valid (no duplicate, no leading-zero concern here). **Choose it.**
    - Then `i=1`, `digits[1]=2`: `len(path)==2`, check evenness → `2` is even → valid.
      - `path = [2,4,2]` → **count += 1** (number: 242)
- Back at `len(path)==0`, try `i=1`: `digits[1]=2`. **Skip check fires!** `digits[1]==digits[0]` and `used[0]` is `False` (we've fully backtracked out of that branch) → **skip**. This correctly avoids re-deriving `224` and `242` a second time via a different starting index.
- `i=2`, `digits[2]=4`: valid as hundreds digit.
  - ... (continues exploring numbers starting with 4)

Notice: without the skip at that last bullet, the algorithm would have re-explored the *entire* subtree starting with hundreds-digit `2` all over again (via `i=1` instead of `i=0`), producing the exact same numbers `224` and `242` a second time — wasted work that a final `set()` would have to clean up after the fact. The skip check prevents this at the source.

---

## Key Takeaways

1. **Track state by index, not value**, whenever duplicate input values can each be independently used.
2. **The "choose → recurse → undo" cycle** is what allows shared mutable state (`path`, `used`) to be safely reused across sibling branches.
3. **Position-based constraints** should be checked against `len(path)` (or a depth counter), separate from the value-based constraints checked against `digits[i]` itself.
4. **Duplicate-skipping requires a sorted array** and the specific `not used[i-1]` condition — this is a reusable pattern worth memorizing, as it appears across many "avoid duplicate combinations/permutations" problems.
5. Getting duplicate-skipping right during generation is more efficient than generating everything and deduplicating with a `set()` afterward — but the `set()` approach is a perfectly valid fallback when constraints are small and you want a simpler, more obviously-correct first pass.