![[Pasted image 20260723233955.png]]
![[Pasted image 20260723234008.png]]


Changing element on a list
![[Pasted image 20260723234044.png]]


list length
![[Pasted image 20260723234120.png]]

delete an item on a list
![[Pasted image 20260723234208.png]]
changed the index of India after deleting Canada
![[Pasted image 20260723234245.png]]

getting the value of the last element on the list
![[Pasted image 20260723234403.png]]




Sure — both use Python's slice notation `list[start:stop:step]`, where leaving `start` and `stop` empty means "the whole list," and `step` controls direction and spacing.

## `list1[::3]`

```python
list1 = [10, 11, 12, 13, 14]
print(list1[::3])
```

- `start` is empty → begin at index 0
- `stop` is empty → go to the end
- `step` is `3` → take every 3rd element

Indexes: `0:10, 1:11, 2:12, 3:13, 4:14`

Starting at index 0, then jumping by 3 each time: index 0, then index 3.

**Output:** `[10, 13]`

## `list1[::-1]`

```python
print(list1[::-1])
```

- `start` and `stop` are empty
- `step` is `-1` → move backwards, one element at a time

When the step is negative, Python actually reads the list from the _end_ to the _beginning_ (since start/stop are unspecified, it defaults to the full range in the reverse direction).

**Output:** `[14, 13, 12, 11, 10]`

This is a common Python idiom for **reversing a list** without using `.reverse()` or `reversed()`.

### Quick intuition for `step`

|step|meaning|
|---|---|
|`1` (default)|every element, forward|
|`2`|every 2nd element, forward|
|`-1`|every element, backward (reversed)|
|`-2`|every 2nd element, backward|

Want me to show what happens if you also set specific `start`/`stop` values with a negative step (e.g. `list1[3:0:-1]`)? That's where people usually get tripped up.