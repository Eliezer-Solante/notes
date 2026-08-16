![[Pasted image 20260724092937.png]]


`.append()`  Method
![[Pasted image 20260724093206.png]]
Inserts an item at the END of the list

`.insert()` Method
![[Pasted image 20260724093256.png]]
inserts an item anywhere on the list as long as it specifies the index of where will it be inserted. 
If it was inserted in the middle, the list will shift to the right.

Swapping items on a list
(Manual)
![[Pasted image 20260724093635.png]]

One-Line
![[Pasted image 20260724093842.png]]


`.sort()` method
![[Pasted image 20260724094535.png]]

`.reverse()` method
![[Pasted image 20260724094609.png]]

Here's a quick rundown of Python list methods:

**`pop(index=-1)`** — Removes and _returns_ the item at the given index (default: last item). Useful when you need the removed value.

```python
lst = [1, 2, 3]
x = lst.pop()      # x = 3, lst = [1, 2]
y = lst.pop(0)     # y = 1, lst = [2]
```

**`remove(value)`** — Removes the _first_ occurrence of a value (not by index). Raises `ValueError` if not found.

```python
lst = [1, 2, 3, 2]
lst.remove(2)      # lst = [1, 3, 2]
```

**`clear()`** — Empties the list completely (in place).

```python
lst = [1, 2, 3]
lst.clear()         # lst = []
```

**`index(value)`** — Returns the index of the first occurrence of a value. Raises `ValueError` if not found. Optional start/end args to limit the search range.

```python
lst = [10, 20, 30]
lst.index(20)        # 1
```

**`count(value)`** — Returns how many times a value appears in the list.

```python
lst = [1, 2, 2, 3, 2]
lst.count(2)          # 3
```

**`extend(iterable)`** — Appends all items from another iterable, one by one (flattens), unlike `append` which adds it as a single element.

```python
lst = [1, 2]
lst.extend([3, 4])     # lst = [1, 2, 3, 4]
```

**`copy()`** — Returns a shallow copy of the list (same as `lst[:]`).

```python
a = [1, 2, 3]
b = a.copy()            # independent copy
```

Quick distinction worth remembering: `pop` returns the removed value, `remove` doesn't return anything meaningful (returns `None`), and `extend` vs `append` is a classic gotcha — `append([3,4])` adds a nested list, `extend([3,4])` adds `3` and `4` individually.

Additional
`max()` isn't actually a list _method_ — it's a built-in Python function that works on any iterable (lists, tuples, strings, etc.) to find the largest item.

**Basic usage:**

```python
lst = [3, 7, 2, 9, 4]
print(max(lst))    # 9
```

**With multiple arguments (not a list):**

```python
print(max(5, 12, 3))   # 12
```

**With strings** — compares alphabetically:

```python
print(max("banana"))         # 'n' (largest letter)
print(max(["apple", "zebra", "mango"]))   # 'zebra'
```

**Using `key`** — customize what "largest" means:

```python
words = ["hi", "hello", "hey"]
print(max(words, key=len))    # 'hello' (longest string)

people = [("Alice", 30), ("Bob", 25), ("Cara", 35)]
print(max(people, key=lambda p: p[1]))   # ('Cara', 35) — oldest
```

**Using `default`** — avoid errors on empty iterables:

```python
empty = []
print(max(empty, default=0))   # 0 (instead of ValueError)
```

Its counterpart is `min()`, which works exactly the same way but finds the smallest item.

`min()` is the counterpart to `max()` — same built-in function, but it returns the _smallest_ item instead of the largest.

**Basic usage:**

```python
lst = [3, 7, 2, 9, 4]
print(min(lst))    # 2
```

**With multiple arguments (not a list):**

```python
print(min(5, 12, 3))   # 3
```

**With strings** — compares alphabetically:

```python
print(min("banana"))          # 'a' (smallest letter)
print(min(["apple", "zebra", "mango"]))   # 'apple'
```

**Using `key`** — customize what "smallest" means:

```python
words = ["hello", "hi", "hey"]
print(min(words, key=len))    # 'hi' (shortest string)

people = [("Alice", 30), ("Bob", 25), ("Cara", 35)]
print(min(people, key=lambda p: p[1]))   # ('Bob', 25) — youngest
```

**Using `default`** — avoid errors on empty iterables:

```python
empty = []
print(min(empty, default=0))   # 0 (instead of ValueError)
```

Same rules apply as `max()`: it works on any iterable, not just lists, and raises `ValueError` if the iterable is empty and no `default` is given.


![[Pasted image 20260724105319.png]]

`.index()` is a list method that finds the **position (index)** of the first occurrence of a given value in a list.

**Basic syntax:**

```python
list.index(value)
```

**Example:**

```python
fruits = ["apple", "banana", "cherry", "banana"]
print(fruits.index("banana"))    # 1
```

It returns `1` because that's the position of the _first_ `"banana"` (Python indexing starts at 0).

---

**If the value isn't in the list**, it raises a `ValueError`:

```python
fruits = ["apple", "banana", "cherry"]
print(fruits.index("mango"))
```

```
ValueError: 'mango' is not in list
```

---

**Optional start/end arguments** — search only within a specific range:

```python
list.index(value, start, end)
```

```python
nums = [10, 20, 30, 20, 40]
print(nums.index(20))         # 1 (first occurrence overall)
print(nums.index(20, 2))      # 3 (start searching from index 2)
```

---

**Quick tip to avoid errors** — check if the value exists first:

```python
fruits = ["apple", "banana", "cherry"]
if "mango" in fruits:
    print(fruits.index("mango"))
else:
    print("Not found")
```

**In short:** `.index()` answers the question _"where is this value in my list?"_ — it returns a position, not the value itself (that's the opposite of what `[]` indexing does, which takes a position and returns the value).