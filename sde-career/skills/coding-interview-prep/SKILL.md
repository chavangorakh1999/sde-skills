---
name: coding-interview-prep
description: "Coding interview preparation: the 7-step problem-solving framework, complexity analysis, common patterns (sliding window, two pointers, BFS/DFS, DP), and how to handle pressure. Use when preparing for or practicing coding interviews."
---

## Coding Interview Preparation

### Context

Problem to solve or prep area: **$ARGUMENTS**

---

### The 7-Step Framework (Every Problem)

```
1. Understand       Read the problem twice. Ask clarifying questions.
2. Examples         Work through 2-3 examples by hand (include edge cases)
3. Brute Force      Describe the O(n²) or naive approach first
4. Optimize         Identify the bottleneck, think of data structures/patterns
5. Code             Write clean code, narrate as you go
6. Test             Trace through your example(s), check edge cases
7. Complexity       State time and space complexity clearly

Time allocation (45-min interview):
  Steps 1-3: 8 min | Step 4: 5 min | Step 5: 20 min | Steps 6-7: 7 min
```

---

### Clarifying Questions (Always Ask)

```
Input:
  - What are the data types? (integers, strings, custom objects?)
  - What's the input range? (array length, value bounds)
  - Can there be negatives? Duplicates? Empty input?

Output:
  - Return format? (index vs. value, any order, unique results)

Constraints:
  - Can I modify the input?
  - Memory constraints?
  - Do I need to optimize for time or space?
```

---

### Pattern Recognition Guide

```
Pattern              Trigger keywords / problem shape
──────────────────────────────────────────────────────────
Sliding Window       "longest subarray/substring with condition X"
                     "minimum window containing..."

Two Pointers         "sorted array, find pair/triplet"
                     "palindrome", "remove duplicates in-place"

Binary Search        "sorted array, find target" (also: "minimum X that satisfies Y")

BFS                  "shortest path", "minimum steps", "level by level"
                     (BFS for shortest, DFS for all paths/existence)

DFS/Backtracking     "all combinations/permutations/subsets"
                     "can we reach X?", "generate all valid..."

Dynamic Programming  "count ways to...", "minimum cost to...", "maximum with constraint"
                     overlapping subproblems + optimal substructure

Heap/Priority Queue  "top K", "kth largest/smallest", "streaming data"

Graph               "number of islands", "connected components", "dependencies"
                    (build adjacency list, then BFS or DFS)

Stack               "valid parentheses", "next greater element", "monotonic"

Hash Map            "two sum", "frequency count", "group by property"
```

---

### Complexity Analysis

```javascript
// TIME COMPLEXITY:
O(1)      — hash map lookup, array index access
O(log n)  — binary search, balanced BST operations
O(n)      — single loop, linear scan
O(n log n)— sort, heap operations over n elements
O(n²)     — nested loops
O(2ⁿ)    — all subsets (exponential)
O(n!)     — all permutations

// SPACE COMPLEXITY:
O(1)      — no extra allocation proportional to input
O(n)      — storing n elements in map/array/stack
O(log n)  — recursive call stack for divide and conquer
O(n²)     — 2D DP table

// State this at the end:
"Time complexity is O(n log n) due to the sort.
 Space complexity is O(n) for the auxiliary hash map."
```

---

### Common Patterns with Code

```javascript
// SLIDING WINDOW — Longest substring with at most K distinct chars
function lengthOfLongestSubstringKDistinct(s, k) {
  const freq = new Map();
  let left = 0;
  let maxLen = 0;

  for (let right = 0; right < s.length; right++) {
    freq.set(s[right], (freq.get(s[right]) ?? 0) + 1);

    while (freq.size > k) {
      const char = s[left++];
      freq.set(char, freq.get(char) - 1);
      if (freq.get(char) === 0) freq.delete(char);
    }

    maxLen = Math.max(maxLen, right - left + 1);
  }
  return maxLen;
}

// BFS — Shortest path in grid
function shortestPath(grid) {
  const rows = grid.length, cols = grid[0].length;
  const dirs = [[0,1],[0,-1],[1,0],[-1,0]];
  const queue = [[0, 0, 1]];   // [row, col, distance]
  const visited = new Set(['0,0']);

  while (queue.length) {
    const [r, c, dist] = queue.shift();
    if (r === rows - 1 && c === cols - 1) return dist;

    for (const [dr, dc] of dirs) {
      const nr = r + dr, nc = c + dc;
      const key = `${nr},${nc}`;
      if (nr >= 0 && nr < rows && nc >= 0 && nc < cols &&
          grid[nr][nc] === 0 && !visited.has(key)) {
        visited.add(key);
        queue.push([nr, nc, dist + 1]);
      }
    }
  }
  return -1;
}

// BINARY SEARCH template
function binarySearch(nums, target) {
  let lo = 0, hi = nums.length - 1;

  while (lo <= hi) {
    const mid = lo + Math.floor((hi - lo) / 2);  // avoid overflow
    if (nums[mid] === target) return mid;
    else if (nums[mid] < target) lo = mid + 1;
    else hi = mid - 1;
  }
  return -1;
}

// DYNAMIC PROGRAMMING — Coin change (minimum coins)
function coinChange(coins, amount) {
  const dp = new Array(amount + 1).fill(Infinity);
  dp[0] = 0;

  for (let i = 1; i <= amount; i++) {
    for (const coin of coins) {
      if (coin <= i) {
        dp[i] = Math.min(dp[i], dp[i - coin] + 1);
      }
    }
  }

  return dp[amount] === Infinity ? -1 : dp[amount];
}
```

---

### Under Pressure

```
If you're stuck:
1. Think out loud — "I know I need to process each element..."
2. Solve a simpler version — "If the array was sorted, I'd..."
3. Think about the data structure — "What if I used a Map here?"
4. Accept a suboptimal start — "Let me start with O(n²) and optimize"

If you find a bug while coding:
  "Actually wait, let me re-check this line..."
  Pause, trace through the example on paper, fix it.
  Don't panic — finding and fixing bugs mid-problem is a green signal.

If the interviewer gives a hint:
  Use it. Say "That's a good point — if I use a sliding window here..."
  Interviewers want to see you incorporate feedback.
```

---

### Study Plan

```
Week 1-2: Array & String fundamentals
  - Two pointers, sliding window, prefix sums
  - Rotate array, valid anagram, longest substring

Week 3-4: Trees & Graphs
  - BFS, DFS, inorder/preorder/postorder
  - Number of islands, clone graph, course schedule

Week 5-6: Dynamic Programming
  - Memoization vs. tabulation
  - Climbing stairs, coin change, longest common subsequence

Week 7-8: Practice under time pressure
  - Random LeetCode medium, 45-min timer
  - Review solutions after

Platform: LeetCode (NeetCode 150 list is excellent)
```
