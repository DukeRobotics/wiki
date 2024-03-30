# Working with Coroutines

This is a guide on how to work with coroutines in Python. Make sure you've read [Introduction to Coroutines](task_planning/intro_to_coroutines.md) before reading this guide.

See [this page](https://docs.python.org/3/reference/datamodel.html#coroutines) for the official Python documentation on coroutines.

## `async` and `await`

In Python, `async` and `await` are used to define native coroutines. The `async` keyword is used to define a coroutine – preferred over generator-based coroutines. The `await` keyword is used to delegate execution to another coroutine or Awaitable object.

Consider the following example:

```python
async def countdown(start):
    print('Countdown start')
    print(start)
    while start > 0:
        await Yield(start)
        start -= 1
        print(start)
    print('Countdown end')
```

In the above example, `countdown` is a native coroutine. When it first runs, it prints 'Countdown start' and the initial value of `start`. It then enters a loop, yielding the value of `start`, decrementing it by 1, and printing the new value. This continues until `start` is 0, at which point it prints 'Countdown end'.

We cannot use the `yield` keyword in a native coroutine. Instead, in task planning, we have defined the Awaitable `Yield` class. Where we would write `yield yield_value` in a generator-based coroutine, we write `await Yield(yield_value)` in a native coroutine to get the same behavior.

This is the definition of the `Yield` class:
```python
class Yield:
    def __init__(self, value):
        self.value = value

    def __await__(self):
        return (yield self.value)
```

We can drive the coroutine using an event loop.

```python
c = countdown(3)
try:
    while True:
        c.send(None)
except StopIteration:
    print('Caught StopIteration')

>>> Countdown start
>>> 3
>>> 2
>>> 1
>>> 0
>>> Countdown end
>>> Caught StopIteration
```
The event loop drives the coroutine until it raises a `StopIteration` exception. This is the signal that the coroutine has completed. It uses the `send` method to drive the coroutine, which is discussed in the next section.

We can also `await` coroutines inside other coroutines. Consider the following example:

```python
async def double_countdown(start1, start2):
    await countdown(start1)
    await countdown(start2)
```

In the above example, `double_countdown` is a coroutine that awaits the `countdown` coroutine twice. The `double_countdown` coroutine will first run the `countdown` coroutine with `start1`, and then run the `countdown` coroutine with `start2`.

```python
dc = double_countdown(3, 5)
try:
    while True:
        dc.send(None)
except StopIteration:
    print('Caught StopIteration')

>>> Countdown start
>>> 3
>>> 2
>>> 1
>>> 0
>>> Countdown end
>>> Countdown start
>>> 5
>>> 4
>>> 3
>>> 2
>>> 1
>>> 0
>>> Countdown end
>>> Caught StopIteration
```

We can see that the `double_countdown` coroutine first runs the `countdown` coroutine with `start1` and then runs the `countdown` coroutine with `start2`. It raises a `StopIteration` exception when both `countdown` coroutines have completed.

## `return` statement

We can `return` a value from a coroutine using the `return` statement. The value returned by the coroutine is the value that the `StopIteration` exception evaluates to.

```python
async def countdown(start):
    print('Countdown start')
    print(start)
    while start > 0:
        await Yield(start)
        start -= 1
        print(start)
    print('Countdown end')
    return 'Countdown complete'

c = countdown(3)
try:
    while True:
        c.send(None)
except StopIteration as e:
    print('Caught StopIteration')
    print(e.value)

>>> Countdown start
>>> 3
>>> 2
>>> 1
>>> 0
>>> Countdown end
>>> Caught StopIteration
>>> Countdown complete
```

## `send` method

The `send` method is used to send a value to the coroutine when it is paused at an `await` expression. It accepts one argument – the value to send to the coroutine. The `await` expression evaluates to the sent value.

Consider the following example:

```python
async def countdown(start):
    print('Countdown start')
    print(start)
    while start > 0:
        step = await Yield(start)
        start -= step
        print(start)
    print('Countdown end')
```

In the above example, the `countdown` coroutine now accepts a value from the `send` method. The value sent to the coroutine is stored in the `step` variable. The coroutine then decrements `start` by `step` instead of 1.

> [!NOTE]
> The first call to the `send` method must be `None`. This is because the coroutine has not yet started executing, and we need to start it before sending any values. The first `send(None)` call will run the coroutine until the first `await` expression that yields.

We can initialize the coroutine and run it until its first `await` expression using the following code:

```python
c = countdown(6)
c.send(None)

>>> Countdown start
>>> 6
```

We can then send values to the coroutine using the `send` method:

```python
c.send(3)
>>> 3

c.send(2)
>>> 1

c.send(1)
>>> 0
>>> Countdown end
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
```

## `throw` method

The `throw` method is used to raise an exception inside the coroutine. It enables the parent to indicate to the coroutine that an exceptional condition has occurred that the coroutine needs to handle. It accepts one argument – an exception object to raise. The exception is raised at the point where the coroutine is paused at an `await` expression.

> [!NOTE]
> There is a second form of the `throw` method that accepts three arguments – the exception type, the exception value, and the traceback. This form is now deprecated and should not be used.

```python
c = countdown(6)
c.send(None)
c.send(3)
c.throw(ValueError)

>>> Countdown start
>>> 6
>>> 3
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ValueError
```

The coroutine can catch the exception using a `try/except` block.

```python
async def countdown(start):
    print('Countdown start')
    print(start)
    while start > 0:
        try:
            step = await Yield(start)
        except ValueError as e:
            print('Caught ValueError')
            start = e.value
        else:
            start -= step
            print(start)
    print('Countdown end')

c = countdown(6)
c.send(None)
c.send(3)
c.throw(ValueError(8))
c.send(0)
c.send(4)

>>> Countdown start
>>> 6
>>> 3
>>> Caught ValueError
>>> 8
>>> 4
```

The parent calls `throw` with a `ValueError` to reset the countdown to a new starting value. The coroutine catches the `ValueError` and sets the value of `start` to the value of the exception. The coroutine then resumes executing when the `send` method is called.

If the coroutine has not yet started executing, the exception is raised at the coroutine header (the line containing the `async def` statement).

```python
c = countdown(6)
c.throw(ValueError)

Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ValueError
```

Here the exception was not caught inside the coroutine, so it was raised at the coroutine header.

## `close` method

The `close` method is used to close the coroutine. Once the coroutine is closed, neither the `send` nor the `throw` method can be used to drive the coroutine.

```python
c = countdown(6)
c.send(None)
c.send(3)
c.close()
c.send(2)

>>> Countdown start
>>> 6
>>> 3
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
RuntimeError: cannot reuse already awaited coroutine
```

Calling `close` raises a `GeneratorExit` exception inside the coroutine, which can be caught using a `try/except` block. This allows the coroutine to perform any cleanup operations before closing.

```python
async def countdown(start):
    print('Countdown start')
    print(start)
    while start > 0:
        try:
            step = await Yield(start)
        except GeneratorExit:
            print('Countdown closed')
            return
        else:
            start -= step
            print(start)
    print('Countdown end')

c = countdown(6)
c.send(None)
c.send(3)
c.close()

>>> Countdown start
>>> 6
>>> 3
>>> Countdown closed
```

> [!ATTENTION]
> When the `close` method is called, the coroutine _must_ end. This happens automatically if the `GeneratorExit` exception is not caught.
>
> If the coroutine catches the `GeneratorExit` exception, it must do one of the following:
> 1. Re-raise the `GeneratorExit` exception
> 2. Raise a new exception
> 3. Run to completion
> If the coroutine does not do any of these, a `RuntimeError` will be raised with message "coroutine ignored GeneratorExit".

In the first and third cases, no exception is raised in the parent even if the coroutine ends (typically this would result in a StopIteration exception). If the coroutine ends by returning a value, that value will _not_ be accessible to the parent (`close()` will always return `None`). However, in the second case, the exception is raised in the parent.

```python
async def countdown(start):
    print('Countdown start')
    print(start)
    while start > 0:
        try:
            step = await Yield(start)
        except GeneratorExit:
            print('Countdown closed')
        else:
            start -= step
            print(start)
    print('Countdown end')

c = countdown(6)
c.send(None)
c.send(3)
c.close()

>>> Countdown start
>>> 6
>>> 3
>>> Countdown closed
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
RuntimeError: coroutine ignored GeneratorExit
>>> Countdown closed
Exception ignored in: <coroutine object countdown at 0x7f8b3c3b3d60>
RuntimeError: coroutine ignored GeneratorExit
```

In the above example, the coroutine catches the `GeneratorExit` exception and prints "Countdown closed". However, the coroutine does not re-raise the `GeneratorExit`, raise a new exception, or run to completion. Instead, it loops back and attempts to `await Yield`. This causes Python to raise a `RuntimeError` with the message "coroutine ignored GeneratorExit".

Since the program ends, the coroutine is then garbage collected. This causes the `__del__` method to be called (see below), attempts to close the coroutine again. This causes "Countdown closed" to be printed again and a `RuntimeError` to be raised again with the message "coroutine ignored GeneratorExit".

It is possible to call `throw` with a `GeneratorExit` exception. However, this does not behave like calling `close`.

```python
async def countdown(start):
    print('Countdown start')
    print(start)
    while start > 0:
        try:
            step = await Yield(start)
        except GeneratorExit:
            print('Countdown closed')
            return start
        else:
            start -= step
            print(start)
    print('Countdown end')

c = countdown(6)
c.send(None)
c.send(3)
print(c.close())

>>> Countdown start
>>> 6
>>> 3
>>> Countdown closed
>>> None

c = countdown(6)
c.send(None)
c.send(3)
c.throw(GeneratorExit)

>>> Countdown start
>>> 6
>>> 3
>>> Countdown closed
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration: 3
```

In the first example, the coroutine is closed using the `close` method. The coroutine catches the `GeneratorExit` exception and prints "Countdown closed". The coroutine then returns the value of `start` (3) to the parent, but the parent does not recieve this value (`None` is printed) and no exception is raised.

In the second example, the coroutine is closed using the `throw` method with a `GeneratorExit` exception. The coroutine catches the `GeneratorExit` exception, prints "Countdown closed", and returns the value of `start` (3). This causes a `StopIteration` exception to be raised in the parent, which evaluates to the value of `start`.

## `__del__` method

The `__del__` method is called when the coroutine is garbage collected. It is a wrapper around the `close` method, which closes the coroutine. Thus, any cleanup operations that the coroutine performs when it `close`'s are also performed when the coroutine is garbage collected.

```python
async def countdown(start):
    print('Countdown start')
    print(start)
    while start > 0:
        try:
            step = await Yield(start)
        except GeneratorExit:
            print('Countdown closed')
            return
        else:
            start -= step
            print(start)
    print('Countdown end')

c = countdown(6)
c.send(None)
c.send(3)
del c

>>> Countdown start
>>> 6
>>> 3
>>> Countdown closed
```

As with the `close` method, if the coroutine doesn't handle the `GeneratorExit` exception properly, when the object is garbage collected, it will result in a `RuntimeError` with the message "coroutine ignored GeneratorExit".

```python
async def countdown(start):
    print('Countdown start')
    print(start)
    while start > 0:
        try:
            step = await Yield(start)
        except GeneratorExit:
            print('Countdown closed')
        else:
            start -= step
            print(start)
    print('Countdown end')

c = countdown(6)
c.send(None)
c.send(3)

>>> Countdown start
>>> 6
>>> 3
>>> Countdown closed
Exception ignored in: <coroutine object countdown at 0x7f8b3c3b3d60>
RuntimeError: coroutine ignored GeneratorExit
```

In the above example, the coroutine does raise an exception or return after catching the `GeneratorExit` exception. When the coroutine is garbage collected, a `RuntimeError` is raised with the message "coroutine ignored GeneratorExit".

## Summary

Native coroutines in Python are defined using the `async` and `await` keywords. The `await` keyword is used to delegate execution to another coroutine or Awaitable object. The `send` method is used to send a value to the coroutine when it is paused at an `await` expression. The `throw` method is used to raise an exception inside the coroutine. The `close` method is used to close the coroutine. The `__del__` method is called when the coroutine is garbage collected and is a wrapper around the `close` method.
