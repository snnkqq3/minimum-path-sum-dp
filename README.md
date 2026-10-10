# Minimum Path Sum: Solve LeetCode 64 With a 2D DP Recurrence, Avoid the Greedy Trap, and Cut Space to O(n)

Most people who search for minimum path sum have already read the problem statement and are stuck somewhere between "I know this is dynamic programming" and "I can write it correctly under interview pressure." The grid is small, the rules are simple, and yet there are three or four places where a nearly-correct solution returns the wrong number.

Here's the full picture: the recurrence, a hand-traced example, working code in Python, Java, and C++, the space optimization interviewers ask about, and the variants that look identical but aren't.

## What the problem actually asks

You get an `m x n` grid of non-negative integers. Start at the top-left cell `(0, 0)`, finish at the bottom-right cell `(m-1, n-1)`, and you may only move **down** or **right**. Return the smallest possible sum of the numbers along such a path. Both endpoints count.

text
grid = [[1,3,1],
        [1,5,1],
        [4,2,1]]

Output: 7
Path: 1 → 3 → 1 → 1 → 1   (right, right, down, down)


Second example: `[[1,2,3],[4,5,6]]` returns `12` via `1 → 2 → 3 → 6`.

Constraints worth noting because they shape your implementation: `1 <= m, n <= 200` and `0 <= grid[i][j] <= 200`. The largest possible answer is roughly `200 × 400 = 80,000`, so a 32-bit `int` is fine — no overflow handling needed. Two details matter more than they look: values can be **zero**, and grids can be a **single row or a single column**.

## Why greedy fails here

The first instinct for a lot of candidates is to walk from `(0,0)` and always step onto the cheaper neighbor. Try it on the LeetCode example:

- At `(0,0)=1`: right is `3`, down is `1` → go down.
- At `(1,0)=1`: right is `5`, down is `4` → go down.
- At `(2,0)=4`: right is `2` → then `1`.

Total: **9**. The real answer is **7**. Greedy commits to a locally cheap cell without knowing what comes after it, and there's no way to un-commit, because the moves only go forward.

That failure mode is exactly why this problem exists in interview question banks. It's a clean test of whether you reach for a state definition instead of a heuristic.

## The state definition and recurrence

Let `f[i][j]` be the minimum path sum from `(0,0)` to cell `(i,j)`, including `grid[i][j]`.

Since a path can only arrive at `(i,j)` from above or from the left:

text
f[i][j] = grid[i][j] + min(f[i-1][j], f[i][j-1])


Boundaries fall out of the move rules:

- `f[0][0] = grid[0][0]`
- First row, `i = 0, j > 0`: only from the left → `f[0][j] = f[0][j-1] + grid[0][j]`
- First column, `j = 0, i > 0`: only from above → `f[i][0] = f[i-1][0] + grid[i][0]`

Answer: `f[m-1][n-1]`.

For context on why brute force isn't an option: the number of monotone paths through an `m x n` grid is `C(m+n-2, m-1)`. At `m = n = 200` that's on the order of `10^118`. Enumerating paths is not a slow solution, it's a theoretically impossible one.

## Tracing the 3×3 example by hand

| Cell | Comes from | Value |
| --- | --- | --- |
| `f[0][0]` | start | 1 |
| `f[0][1]` | left 1 + 3 | 4 |
| `f[0][2]` | left 4 + 1 | 5 |
| `f[1][0]` | above 1 + 1 | 2 |
| `f[1][1]` | min(4, 2) + 5 | 7 |
| `f[1][2]` | min(5, 7) + 1 | 6 |
| `f[2][0]` | above 2 + 4 | 6 |
| `f[2][1]` | min(7, 6) + 2 | 8 |
| `f[2][2]` | min(6, 8) + 1 | **7** |

Filled table:

text
[1, 4, 5]
[2, 7, 6]
[6, 8, 7]


`f[2][2] = 7`, and tracing the choices backwards recovers the path `right, right, down, down`.

## Three implementations

### 1. Top-down with memoization

Useful as a "correct but slower" first answer in an interview, because you can derive it from brute-force recursion in one sentence.

python
from functools import lru_cache

def minPathSum(grid):
    rows, cols = len(grid), len(grid[0])

    @lru_cache(maxsize=None)
    def best(r, c):
        if r == 0 and c == 0:
            return grid[0][0]
        if r == 0:
            return best(0, c - 1) + grid[r][c]
        if c == 0:
            return best(r - 1, 0) + grid[r][c]
        return grid[r][c] + min(best(r - 1, c), best(r, c - 1))

    return best(rows - 1, cols - 1)


Two traps to avoid in this version. First, don't write `grid[r][c] + min(best(r-1, c), best(r, c-1))` without the `r == 0` / `c == 0` guards and expect negative-index wraparound to save you — Python's `grid[-1]` silently reads the last row. Second, if you memoize into an array initialized to `0`, remember that `0` is a legal partial sum here. Use `-1` or a separate `visited` structure.

### 2. Bottom-up 2D on an explicit table

This is the version most interviewers expect, and the one that maps directly onto the table above.

java
class Solution {
    public int minPathSum(int[][] grid) {
        int rows = grid.length, cols = grid[0].length;
        int[][] f = new int[rows][cols];
        f[0][0] = grid[0][0];

        for (int i = 1; i < rows; i++) f[i][0] = f[i - 1][0] + grid[i][0];
        for (int j = 1; j < cols; j++) f[0][j] = f[0][j - 1] + grid[0][j];

        for (int i = 1; i < rows; i++)
            for (int j = 1; j < cols; j++)
                f[i][j] = Math.min(f[i - 1][j], f[i][j - 1]) + grid[i][j];

        return f[rows - 1][cols - 1];
    }
}


cpp
int minPathSum(vector<vector<int>>& grid) {
    int rows = grid.size(), cols = grid[0].size();
    vector<vector<int>> f(rows, vector<int>(cols, 0));
    f[0][0] = grid[0][0];

    for (int i = 1; i < rows; ++i) f[i][0] = f[i - 1][0] + grid[i][0];
    for (int j = 1; j < cols; ++j) f[0][j] = f[0][j - 1] + grid[0][j];

    for (int i = 1; i < rows; ++i)
        for (int j = 1; j < cols; ++j)
            f[i][j] = min(f[i - 1][j], f[i][j - 1]) + grid[i][j];

    return f[rows - 1][cols - 1];
}


Skipping the two boundary loops is the single most common bug. It only shows up on grids that are a single row or single column, which is precisely what a good interviewer tests.

### 3. Rolling array: O(n) space

Row `i` needs only row `i-1`. So collapse the table to a single 1D array of length `cols`, where before you update index `j`, `dp[j]` still holds `f[i-1][j]` (from above), and `dp[j-1]` has already been overwritten with `f[i][j-1]` (from the left):

python
def minPathSum(grid):
    rows, cols = len(grid), len(grid[0])
    dp = [0] * cols
    dp[0] = grid[0][0]
    for j in range(1, cols):
        dp[j] = dp[j - 1] + grid[0][j]

    for i in range(1, rows):
        dp[0] += grid[i][0]          # first column: only from above
        for j in range(1, cols):
            dp[j] = min(dp[j], dp[j - 1]) + grid[i][j]

    return dp[-1]


The iteration order is load-bearing. `j` must run left to right, and the left-to-right direction is what makes `dp[j-1]` mean "current row" while `dp[j]` still means "previous row." Reverse it and the recurrence reads garbage.

> Space complexity drops to `O(n)`. If you also want to shrink it in the other dimension, transpose the grid when `rows < cols` so the rolling array follows the shorter side — `O(min(m, n))` space.

### Complexity, side by side

| Approach | Time | Space |
| --- | --- | --- |
| Naive DFS over all paths | `O(2^(m+n))` | `O(m+n)` recursion depth |
| Top-down with memoization | `O(m·n)` | `O(m·n)` + recursion stack |
| Bottom-up 2D table | `O(m·n)` | `O(m·n)` |
| Bottom-up rolling array | `O(m·n)` | `O(n)`, or `O(min(m,n))` with transpose |

A note on in-place solutions: you can overwrite `grid[i][j]` with `f[i][j]` and give up the extra array entirely, but confirm with the interviewer first. Mutating an input argument is a real change in behavior, and some interviewers treat it as a correctness issue rather than an optimization.

## Variants that share this skeleton

Getting minimum path sum right is mostly valuable because the same two-row recurrence reappears in a family of problems. What changes is usually just the move set or the constraint.

- **Unique Paths (#62)** — identical movement, but you count paths instead of summing costs, so the recurrence becomes `f[i][j] = f[i-1][j] + f[i][j-1]`. The combinatorics shortcut `C(m+n-2, m-1)` works there and does *not* carry over to #64, because costs break the symmetry between paths.
- **Unique Paths II (#63)** — add obstacles. Treat an obstacle cell as unreachable: keep `f[i][j] = 0` for counting, or skip it for minimization.
- **Minimum Falling Path Sum (#931)** and **Triangle (#120)** — the predecessor set grows from two cells to three. The structure is unchanged; the minimum is simply taken over more neighbors.
- **Dungeon Game (#174)** — this one looks like minimum path sum and breaks the forward recurrence, because the constraint is that health must stay positive at *every* step rather than that costs simply add up. It needs a reverse DP. If you try to force the #64 recurrence onto it, you'll get an answer that looks plausible and isn't.

The recognition cue for the whole family: a grid, movement restricted to a monotone set of directions, and a question asking for an extremum over paths. Dynamic programming is the answer essentially every time.

## Where to practice this pattern properly

Minimum path sum on its own takes about twenty minutes to learn. The thing that actually costs people offers is not being able to tell, in the first three minutes of a 45-minute screen, that the problem in front of them is a 2D DP with `f[i][j]` depending on `f[i-1][j]` and `f[i][j-1]`.

AlgoMonster is built by ex-Google engineers around exactly that gap. Its structure is 48 coding patterns, 231 lessons and problems, and roughly 700 diagrams — text and interactive diagrams rather than video, which is either a feature or a dealbreaker depending on how you learn. Each pattern ships with a reusable code template in Python, Java, JavaScript, C++, Go, and Haskell, plus decision flowcharts for the "which pattern is this?" step. Its dynamic programming track runs from 1D DP into 2D grid DP, and problem #64 gets a step-by-step table walkthrough of the same fill order used above. There's a free practice tool and flowchart you can use before paying anything.

Two honest caveats from third-party reviews: the problem bank is around 230 questions versus LeetCode's 3,000+, so most people still use it alongside LeetCode for volume; and system design coverage is introductory, not enough for senior-level loops on its own.

## AlgoMonster pricing, plan by plan

AlgoMonster has no monthly option. You either pay annually or buy lifetime access, and the free tier gives limited access to a subset of patterns and problems — enough to judge whether the text-and-diagram format works for you, not enough to prep for interviews.

| Plan | What you get | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems; free practice tool and flowchart | $0 | No card needed | [Start with the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems, unlimited AI assistant, unlimited Speedrun, company question sets updated monthly, 12 months of full access, bonus Foundation and "Build an LLM" courses | $99/year (about $8.25/month, $0.28/day) | Billed annually, cancel or upgrade anytime | [See the Pro-Yearly plan](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan plus lifetime updates, paid once | $189 | One-time payment | [Check the Pro-Lifetime price](https://algo.monster?fpr=li69) |

Prices here move with promotions, and the numbers you'll see quoted around the web are all over the place — $89, $99, $299, and $459 lifetime all appear in reviews, and official checkout pages have shown both $69.99 and $99 for the annual plan in different crawls. Treat the checkout page as the only authoritative number, and check it right before you buy rather than trusting a comparison table (this one included) written at some earlier date.

Which plan makes sense depends on your timeline. If you have an interview in six weeks, the annual plan covers the whole 4–8 week curriculum window and you'll never touch the lifetime option again. If you expect to switch jobs two or three times over the next several years and revisit interview prep each time, lifetime access pays for itself against a second annual renewal. Note also that there's no formal free trial or enterprise plan, and the refund policy is limited, so spend a few days inside the free tier first — enough to confirm that reading diagrams instead of watching videos actually works for your brain.

If minimum path sum is where your DP prep starts, that's a reasonable place to start. Working through the 2D grid family in order — #62, #63, #64, #120, #931 — and finishing with #174 to see where the whole approach stops working will teach you more than solving thirty unrelated problems.
