### 🪐 Using Jupyter Notebook for Your Exercises

You will be completing the actual practical assessment on a **Jupyter Notebook environment**, so we strongly recommend doing these practice exercises the same way. It'll make you much more comfortable navigating the interface on assessment day.

**Access your Jupyter environment via KodeKloud Playgrounds:**

1. Log in to KodeKloud and launch the [Jupyter Notebook playground](https://learn.kodekloud.com/user/playgrounds/playground-jupyter)
2. Write and run your solutions directly in the notebook
3. Once you're satisfied with your solution, copy it into the corresponding `solution.py` file in your repo and commit your work.



cd ~
python3.9 -m venv venv39
source venv39/bin/activate
pip install jupyter

### 📝Practical Assessment Guidelines

You will always be given a function per Task:

> ```python
> def add_numbers(a: int, b: int):
>     pass
> ```

To ensure your solution is graded correctly,

- Write your code and all logic needed to solve the problem **inside the provided function.**
    
- Keep the function name and parameters exactly as given.
    
- Ensure that the required value being asked has the correct format. 
    

We will execute **only the provided function**, so keeping your solution self-contained inside it ensures your werk is evaluated properly.

###### **⚠️ Avoid**

Placing logic outside the function:

> ```python
> result = 3 + 5
> 
> def add_numbers(a: int, b: int):
>     pass
> ```

In this case, the checker runs:

> ```python
> add_numbers(3, 5)
> ```

But the function returns nothing, so it will not pass.

###### **✅ Correct Approach**

All logic is placed inside the function, and the result is returned:

> ```python
> def add_numbers(a: int, b: int):
>     result = a + b
>     return result
> ```

So when the checker runs:

> ```python
> add_numbers(3, 5)
> ```

it receives the expected outcome:

> ```
> 8
> ```