![[Pasted image 20260724115928.png]]

examples:
This code builds a 2D list (a matrix) using a **nested list comprehension**, then accesses one specific element from it.

## Step 1: Breaking down the comprehension

```python
matrix = [[j for j in range(3)] for i in range(3)]
print (matrix[2][1])
```

Think of this as an **outer comprehension** wrapping an **inner comprehension**:

- **Outer part:** `[... for i in range(3)]` — runs 3 times (creates 3 rows)
- **Inner part:** `[j for j in range(3)]` — runs for each row, creating `[0, 1, 2]`

Since the inner list `[0, 1, 2]` doesn't depend on `i` at all, the **same row gets repeated 3 times**:

```python
matrix = [
    [0, 1, 2],   # i = 0 → inner comprehension builds [0, 1, 2]
    [0, 1, 2],   # i = 1 → inner comprehension builds [0, 1, 2]
    [0, 1, 2],   # i = 2 → inner comprehension builds [0, 1, 2]
]
```

So `matrix` ends up as:

```python
[[0, 1, 2], [0, 1, 2], [0, 1, 2]]
```

## Step 2: Accessing `matrix[2][1]`

This uses **double indexing** — first index picks the row, second picks the column within that row.

```python
matrix[2]       # picks row at index 2 → [0, 1, 2]
matrix[2][1]    # then picks index 1 from that row → 1
```

Visually:

```
        col0  col1  col2
row0:   [0,    1,    2]
row1:   [0,    1,    2]
row2:   [0,    1,    2]
              ↑
        matrix[2][1] = 1
```

## Output

```python
print(matrix[2][1])   # 1
```

## Key takeaway

- `i` (outer loop) never actually gets used inside the inner list — it's only there to control **how many rows** get created.
- `j` (inner loop) is what actually fills each row's values.
- `matrix[row][col]` is how you access elements in a 2D list — first bracket picks the row (a list), second bracket picks an item within that row.

**If `i` had been used**, you'd get a more "real" matrix — for example:

```python
matrix = [[i + j for j in range(3)] for i in range(3)]
# [[0, 1, 2], [1, 2, 3], [2, 3, 4]]
```

Here each row is different because `i` changes per row and gets combined with `j`.