
Average age in the list
![[Pasted image 20260724101323.png]]
In `for age in ages:`, the variable `age` stores the **actual value** of each element, not its index/position.

```python
ages = [56, 72, 24, 46]
for age in ages:
    print(age)
```

Output:

```
56
72
24
46
```

Notice it prints the values themselves (56, 72, 24, 46) — not 0, 1, 2, 3.

**If you wanted the index instead**, you'd need something like:

```python
for i in range(len(ages)):
    print(i)        # prints 0, 1, 2, 3
```

**Or if you want both index and value together**, use `enumerate()`:

```python
for i, age in enumerate(ages):
    print(i, age)
```

Output:

```
0 56
1 72
2 24
3 46
```

So the general pattern `for item in some_list:` always gives you each **element's value**, one at a time — that's just how Python's `for` loop works over iterables like lists.


Nested loop using List
This is a **nested for loop** — a loop inside another loop.

```python
for x in [0, 2, 1, 3]:     
    for y in [0, 4, 1, 2]:             
        print('*')
```

**How it works:**

- The **outer loop** runs once for each item in `[0, 2, 1, 3]` — that's 4 iterations.
- For **each** outer iteration, the **inner loop** runs completely through `[0, 4, 1, 2]` — that's also 4 iterations.
- `print('*')` runs once every time the inner loop executes.

**Important:** the actual _values_ of `x` and `y` don't matter here at all — they're never used inside the loop body. Only the _number of items_ in each list matters, since that determines how many times each loop runs.

**Total iterations = outer × inner = 4 × 4 = 16**

So this prints `*` sixteen times:

```
*
*
*
*
*
*
*
*
*
*
*
*
*
*
*
*
```

**Think of it like a grid:** for every single value of `x` (4 of them), you sweep through _all_ values of `y` (4 of them) before moving to the next `x`. That's the classic nested-loop pattern — useful for things like generating grids, comparing every pair of items, or building multiplication tables.

If `x` and `y` _were_ used inside the loop (e.g., `print(x, y)`), you'd see all 16 combinations of the two lists printed out.

![[Pasted image 20260724103347.png]]
