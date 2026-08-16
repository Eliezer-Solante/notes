### Say "Hello, World!" With Python
```
if __name__ == '__main__':
    my_string= "Hello, World!"
    print(my_string)
```
Result
```
Hello, World!
```

### Python If-Else
Task:
![[Pasted image 20260725162254.png]]

```
#!/bin/python3

import math
import os
import random
import re
import sys

if __name__ == '__main__':
    n = int(input().strip())

    if n % 2 != 0:
        print("Weird")
    elif n in range(2, 6):
        print("Not Weird")
    elif n in range(6, 21):
        print("Weird")
    else:
        print("Not Weird")
```
Sample Input
```
3
```
Sample Output
```
Weird
```


### Arithmetic Operators
Task:
![[Pasted image 20260725162337.png]]

```
if __name__ == '__main__':
    a = int(input())
    b = int(input())

    sums = a + b
    diff = a - b
    prod = a * b
    
    print(sums)
    print(diff)
    print(prod)
```
Sample Input
```
3
2
```
Sample Output
```
5
1
6
```


### Python: Division
Task:
![[Pasted image 20260725162431.png]]

```
if __name__ == '__main__':
    a = int(input())
    b = int(input())

    div = a // b
    fdiv = a/b

    print(div)
    print(fdiv)
```
Sample Input
```
4
3
```
Sample Output
```
1
1.33333333333
```


### Loops
Task:
![[Pasted image 20260725162510.png]]
![[Pasted image 20260725162650.png]]

```
if __name__ == '__main__':
    n = int(input())
    i = 0
    while i != n:
        print(i**2)
        i += 1
```
Sample Input
```
5
```
Sample Output
```
0
1
4
9
16
```


### Lists
Task:
![[Pasted image 20260725162723.png]]
```
if __name__ == '__main__':
    N = int(input())
    list =[]

    for _ in range(N):
        parts = input().split()
        meth = parts[0]
        if meth == "insert":
            list.insert(int(parts[1]), int(parts[2]))
        elif meth == "remove":
            list.remove(int(parts[1]))
        elif meth == "append":
            list.append(int(parts[1]))
        elif meth == "sort":
            list.sort()
        elif meth == "pop":
            list.pop()
        elif meth == "reverse":
            list.reverse()
        elif meth == "print":
            print(list)
        else:
            print("invalid method")
```
Sample Input
```
12
insert 0 5
insert 1 10
insert 0 6
print
remove 6
append 9
append 1
sort
print
pop
reverse
print
```
Sample Output
```
[6, 5, 10]
[1, 5, 9, 10]
[9, 5, 1]
```


### Swap Case
Task:
![[Pasted image 20260725162823.png]]
```
def swap_case(s):
    swapped = ""
    for i in s:
        if i.isupper():
            result = i.lower()
            swapped += result
        elif i.islower:
            result = i.upper()
            swapped += result
    return swapped

if __name__ == '__main__':
    s = input()
    result = swap_case(s)
    print(result)
```
Sample Input
```
HackerRank.com presents "Pythonist 2".
```
Sample Output
```
hACKERrANK.COM PRESENTS "pYTHONIST 2".
```


### String Split and Join
Task:
![[Pasted image 20260725163006.png]]
```
def split_and_join(line):
    splitted = line.split(" ")
    joined = "-".join(splitted)
    return joined

if __name__ == '__main__':
    line = input()
    result = split_and_join(line)
    print(result)
```
Sample Input
```
this is a string
```
Sample Output
```
this-is-a-string
```


### Mutations
Task:
![[Pasted image 20260725163122.png]]

```
def mutate_string(string, position, character):
    l = list(string)
    l[position] = character
    newString = ''.join(l)
    return newString

if __name__ == '__main__':
    s = input()
    i, c = input().split()
    s_new = mutate_string(s, int(i), c)
    print(s_new)
```
Sample Input
```
STDIN           # Function
-----           # --------
abracadabra     # s = 'abracadabra'
5 k             # position = 5, character = 'k'
```
Sample Output
```
abrackdabra
```


### Introduction to Sets
Task:
![[Pasted image 20260725163350.png]]
Note:
![[Pasted image 20260725163413.png]]

```
def average(array):
    setArr = set(array)
    average = sum(setArr) / len(setArr)
    answer = round(average, 10)
    return answer

if __name__ == '__main__':
    n = int(input())
    arr = list(map(int, input().split()))
    result = average(arr)
    print(result)
```
Sample Input
```
STDIN                                       # Function
-----                                       # --------
10                                          # arr[] size N = 10
161 182 161 154 176 170 167 171 170 174     # arr = [161, 181, ..., 174]
```
Sample Output
```
169.375
```


### Exceptions
Task:
![[Pasted image 20260725163528.png]]

```
T = int(input())
for _ in range(T):  
    try:
        parts = input().split()
        dividend = int(parts[0])
        divisor = int(parts[1])
        ans = dividend // divisor        
    except (ZeroDivisionError, ValueError) as e:
        print("Error Code:", e)
    else: 
        print(ans)
```
Sample Input
```
3
1 0
2 $
3 1
```
Sample Output
```
Error Code: integer division or modulo by zero 
Error Code: invalid literal for int() with base 10: '$' 
3
```



### Words Score
Task:
![[Pasted image 20260725163716.png]]
```
def score_words(word, word_count):
    score = 0
    valid_letters = ("a", "e", "i", "o", "u", "y")
    for i in range(1):
        for j in range(word_count):
            vletter_count = 0
            for k in word[j]:
                if k in valid_letters:
                    vletter_count += 1
            if vletter_count % 2 == 0:
                score += 2
            else:
                score += 1                    
    return score

  

if __name__ == '__main__':
    n = int(input())
    word = input().split()
    result = score_words(word, n)
    print(result)
```
Sample Input
```
2
hacker book
```
Sample Output
```
4
```
![[Pasted image 20260725163731.png]]


### Capitalize!
```

```
Sample Input
```

```
Sample Output
```

```



### Integers Come In All Sizes
```

```
Sample Input
```

```
Sample Output
```

```






LIST COMPREHENSION

python
```python
List comprehension =[expression for value in iterable if condition]
```


```python
[w.upper() if len(w) > 3 else w for w in words]
```
List comprehension =[expression for value in iterable if condition]
Equivalent to:

python

```python
result = []
for w in words:
    if len(w) > 3:
        result.append(w.upper())
    else:
        result.append(w)
```