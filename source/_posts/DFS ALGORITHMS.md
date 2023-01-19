---
title: Depth First Search (DFS) Algorithms
date: 2022-12-16 14:52:44
updated: 2022-12-16 14:52:44
tags:
  - '笔记'
categories:
  - 'Algorithm'
  - 'Tree'
  - 'DFS'
lang: en
---

## Intuition

DFS is a recursive algorithm that use backtracking principle for graph traversal.

The main intuition for DFS is:

1. Find the beginning vertex of the graph, marked as visited and output, and try to get into its next options.
2. If this option has child vertex then try to get into next option.
3. Until arrive at the bottom vertex which don't have visited child vertex.
4. Then Track back to the **last** vertex, if it has unvisited options, get into the option and do this four steps again;
5. Until all vertexes have been visited.

Here is an example.

```text
  1 - 4     8 - 9
 /| \      /|
0 |   3 - 5 |
 \|      / \|
  2     6   7
```

the order of next graph visited is:
**0** -> **1** -> **4** -> 1 -> **3** -> **5** -> **8** -> **9** -> 8 -> **7** -> 8 -> 5 -> **6** -> 5 -> 3 -> 1 -> **2** -> 1 -> 0

the order of the out put is: 0 -> 1 -> 4 -> 3 -> 5 -> 8 -> 9 -> 7 -> 6 -> 2

## Implementation

```text
  1
 / \
0 - 3 - 4
 \ /
  2
```

### Recursion

The intuition inspire us that for each vertex we can ignore the last one and do same about the next one. So we can use `Dfs(G, v)` means (graph, vertex), recursive to each vertex.

In the function, we do `visit()`, `marked()` to current vertex and recurse its all next unvisited vertex.

```js
function Dfs(G, v) {
  visit(v); // at the begin
  v.marked = true;
  for (let w in v.neighbors) {
    if (w.marked === false) Dfs(G, w);
  }
}
```

This is a preorder method. There is also a different order for Dfs called postoder that print out the vertex after all its sub vertex have been visited.

```js
function Dfs(G, v) {
  v.marked = true;
  for (let w in v.neighbors) {
    if (w.marked === false) Dfs(G, w);
  }
  visit(v); // at the end
}
```

### Stack

We also can use a stack data structure to implement it.

We pop the vertex in the stack one by one to visit it. And if it has unvisited vertex, we push all the unvisited child into stack waiting to visit. And continue popping in loop.

```js
function Dfs(G, v) {
  const stack = [v];
  while (stack.length > 0) {
    const v = stack.pop();
    if (v.marked === false) {
      visit(v);
      v.marked === true;
    }
    for (let w in v) {
      if (w.marked === false) {
        stack.push(w);
      }
    }
  }
}
```
