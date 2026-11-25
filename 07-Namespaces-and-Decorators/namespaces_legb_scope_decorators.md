# Python Namespaces, LEGB Scope Rule & Decorators

A structured, interview-ready reference covering how Python resolves variables, maintains state via closures, and modifies function behavior using decorators.

---

## 1. Namespaces in Python

A **namespace** is an internal dictionary-like mapping (`{name: object}`) that ensures variable and function names stay isolated and do not collide across different parts of a program.

### Types of Namespaces (Ordered by Lifetime)
1. **Built-in Namespace:** Created when the Python interpreter starts up and exists until it exits. Contains built-in functions and exceptions (`len`, `print`, `int`, `ValueError`).
2. **Global Namespace:** Created when a module/script is imported or executed. Holds module-level variables, functions, and classes.
3. **Enclosing (Nonlocal) Namespace:** Exists only for nested functions—the local scope of an outer function relative to an inner function.
4. **Local Namespace:** Created when a function is called and destroyed as soon as the function returns or raises an unhandled exception.

---

## 2. The LEGB Scope Resolution Rule

When you reference a variable name in Python, the interpreter searches namespaces in strict **LEGB** order and stops at the **first match**:

$$\text{Local (L)} \longrightarrow \text{Enclosing (E)} \longrightarrow \text{Global (G)} \longrightarrow \text{Built-in (B)}$$

If the name is not found in any of the four scopes, Python raises a `NameError`.

### LEGB Example
```python
x = "global"

def outer():
    x = "enclosing"
    
    def inner():
        x = "local"
        print(x)  # Resolves to 'local' (L)
        
    inner()

outer()
```

### Modifying Outer Scopes (`global` & `nonlocal`)
By default, assigning to a variable inside a function creates a **new local variable**. To mutate an outer variable instead:
* **`global var`:** Binds `var` inside a function to the module-level (Global) namespace.
* **`nonlocal var`:** Binds `var` inside a nested function to the nearest Enclosing function namespace (cannot bind to Global).

```python
def counter():
    count = 0
    def increment():
        nonlocal count
        count += 1
        return count
    return increment
```

---

## 3. Closures (The Engine Behind Decorators)

A **closure** occurs when a nested function remembers and accesses variables from its **enclosing scope** even after the outer function has finished executing and its stack frame has been destroyed.

### Three Conditions for a Closure
1. There must be a **nested function** (a function inside a function).
2. The nested function must **reference a variable** defined in the enclosing function.
3. The enclosing function must **return the nested function**.

```python
def multiplier(factor):
    def multiply(number):
        return number * factor  # 'factor' is captured in the closure
    return multiply

double = multiplier(2)
print(double(5))  # Output: 10 (even though multiplier() has finished)
```

---

## 4. Decorators

A **decorator** is a callable (typically a function) that takes another function as an argument, wraps it with additional behavior (logging, timing, authentication, caching), and returns a new function—without modifying the original function's source code.

### Syntactic Sugar (`@decorator`)
Writing:
```python
@my_decorator
def greet():
    print("Hello!")
```
is exact syntactic sugar for:
```python
def greet():
    print("Hello!")

greet = my_decorator(greet)
```

### Production-Grade Decorator Template
A complete decorator should:
1. Accept arbitrary positional and keyword arguments via `*args, **kwargs`.
2. Preserve the original function's return value (`res = func(*args, **kwargs)`).
3. Preserve the original function's metadata (`__name__`, `__doc__`) using `@functools.wraps(func)`.

```python
import time
from functools import wraps

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        elapsed = time.time() - start
        print(f"[TIMER] {func.__name__} executed in {elapsed:.4f}s")
        return result
    return wrapper

@timer
def compute_square(n: int) -> int:
    """Returns the square of n after a short delay."""
    time.sleep(1)
    return n ** 2

print(compute_square(5))
print(compute_square.__name__)  # Prints 'compute_square' (preserved by @wraps)
```

---

## 5. Decorators with Arguments (3-Layer Nesting)

To pass parameters to a decorator itself (e.g., `@repeat(times=3)`), add one extra outer factory layer that accepts the decorator arguments and returns the actual decorator:

```python
from functools import wraps

def repeat(times: int):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            result = None
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(times=3)
def say_hi(name: str):
    print(f"Hi, {name}!")

say_hi("Govinda")
```

---

## 6. Quick Interview Takeaways
* **`UnboundLocalError` Trap:** Reading a global variable inside a function works fine, *unless* you also assign to that variable anywhere later in the same function without `global`—Python marks the variable as local for the entire function body at compile time.
* **Why `@functools.wraps` matters:** Without it, debugging, logging, and automated documentation tools see `wrapper` instead of the decorated function's real `__name__` and `__doc__`.
* **Stacking Decorators:** When multiple decorators are stacked, they apply **bottom-up** (`@A` above `@B` above `def f()` equals `f = A(B(f))`) and execute **top-down** when called.