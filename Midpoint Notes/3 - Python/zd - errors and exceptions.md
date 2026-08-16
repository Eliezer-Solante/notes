![[Pasted image 20260724174125.png]]

## Two main categories

**1. Syntax Errors** — mistakes in code structure, caught _before_ the program runs.

```python
if True
    print("hi")     # missing colon
```

```
SyntaxError: expected ':'
```

**2. Exceptions** — errors that occur _during_ execution, even if the syntax is valid.

```python
print(10 / 0)
```

```
ZeroDivisionError: division by zero
```

## Common built-in exceptions

|Exception|Cause|
|---|---|
|`ValueError`|Right type, invalid value — e.g. `int("abc")`|
|`TypeError`|Wrong type for an operation — e.g. `"5" + 5`|
|`IndexError`|List index out of range — e.g. `[1,2][5]`|
|`KeyError`|Missing dictionary key — e.g. `{"a":1}["b"]`|
|`ZeroDivisionError`|Dividing by zero|
|`NameError`|Using an undefined variable|
|`AttributeError`|Calling a method that doesn't exist — e.g. `[1,2].upper()`|
|`FileNotFoundError`|Opening a file that doesn't exist|
|`UnboundLocalError`|Using a local variable before it's assigned|

## Handling exceptions: `try` / `except`

Instead of letting the program crash, you can catch and handle the error:

```python
try:
    x = 10 / 0
except ZeroDivisionError:
    print("Can't divide by zero!")
```

```
Can't divide by zero!
```

## Full structure

```python
try:
    # risky code
    ...
except SomeError:
    # runs if that specific error occurs
    ...
else:
    # runs only if NO exception occurred
    ...
finally:
    # always runs, error or not (cleanup code)
    ...
```

## Catching multiple exception types

```python
try:
    num = int(input("Enter a number: "))
    print(10 / num)
except ValueError:
    print("That's not a valid number")
except ZeroDivisionError:
    print("Can't divide by zero")
```

## Raising your own exceptions

```python
def check_age(age):
    if age < 0:
        raise ValueError("Age cannot be negative")
    return age
```

**Key takeaway:** syntax errors mean your code is written incorrectly and won't even start running; exceptions mean your code is written correctly but hits a problem while executing — and `try`/`except` lets you handle those problems gracefully instead of crashing the program.