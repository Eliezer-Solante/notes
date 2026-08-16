![[Pasted image 20260724133416.png]]

![[Pasted image 20260724133504.png]]


Dont
![[Pasted image 20260724133529.png]]

default value of the argument
![[Pasted image 20260724133620.png]]
if a value is passed it will be overwritten
![[Pasted image 20260724133707.png]]

The `*` before a parameter name means **"pack all the extra positional arguments into a tuple."** This is called `*args` (though here it's named `friends`, the `*` is what matters, not the name).

## What it does

```python
def my_function(*friends):
    print("The tallest student is " + friends[0])

my_function("john", "Ella", "mark")
```

When you call `my_function("john", "Ella", "mark")`, instead of needing three separate parameters like `def my_function(a, b, c)`, the `*friends` **collects all the arguments you pass in, no matter how many, into a single tuple**.

Inside the function:

```python
friends == ("john", "Ella", "mark")     # a tuple!
```

So:

```python
friends[0]    # "john"
```

Output:

```
The tallest student is john
```

## Why use `*`? — it allows a variable number of arguments

Without `*`, you'd be locked into an exact number of parameters:

```python
def my_function(a, b, c):        # must always pass exactly 3 arguments
    ...
```

With `*friends`, you can pass **any number** of arguments — 1, 3, 10, or even 0:

```python
my_function("john")                          # friends = ("john",)
my_function("john", "Ella")                    # friends = ("john", "Ella")
my_function("john", "Ella", "mark", "Sam")      # friends = ("john", "Ella", "mark", "Sam")
```

## You can loop through `*args` like any iterable

```python
def my_function(*friends):
    for name in friends:
        print(name)

my_function("john", "Ella", "mark")
```

```
john
Ella
mark
```

## Quick comparison

|Without `*`|With `*`|
|---|---|
|`def f(a, b, c):`|`def f(*args):`|
|Must pass exactly 3 arguments|Can pass any number of arguments|
|Access via `a`, `b`, `c`|Access via `args[0]`, `args[1]`, etc., or loop|

## Note on the name `friends`

The `*` is what creates this "gather extra args into a tuple" behavior — the parameter name (`friends`, `args`, `nums`, whatever) is just a regular variable name you choose. `*args` is only a **convention**, not a required name.

**In short:** `*friends` means _"however many arguments get passed in, collect them all into one tuple called `friends`."_ That's why `friends[0]` works — it's just indexing into that tuple like any other sequence.