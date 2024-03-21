# Coroutines

This page serves as an introduction to coroutines in Python. Coroutines are a powerful tool for writing concurrent code, and they are used extensively in the task planning system for the robot.

It covers the history of coroutines in Python, the basics of how they work, and how to use them. It also provides some examples of how coroutines can be used in practice.

The official Python documentation on coroutines can be found [here](https://docs.python.org/3/library/asyncio-task.html). However the following content is intended to be a more comprehensive and accessible introduction to the topic.

The following content is inspired heavily by the following articles. Many of the examples and explanations are taken directly from these articles, and they are highly recommended for further reading:
- [_From yield to async/await_ by mleue](https://mleue.com/posts/yield-to-async-await)
- [_How the heck does async/await work in Python 3.5?_ by Brett Cannon](https://snarky.ca/how-the-heck-does-async-await-work-in-python-3-5/)

> [!NOTE]
> While an introduction to coroutines in Python is typically paired with an introduction to `asyncio`, use of `asyncio` is _not required_ to use coroutines in Python. In fact, the `asyncio` library is _not_ used in the task planning system for the robot and is _not_ covered in this article.

## What is a Coroutine?

A coroutine is a function that can pause and resume its execution. This is useful for writing concurrent code, where you want to be able to do something else while waiting for some operation to complete, or to periodically yield control back to its parent.

Each time a coroutine pauses, it yields control back to its parent. Thus, the parent can then decide what to do next. It can either resume the coroutine that yielded control, or it can execute other code and optionally resume the coroutine later.

The coroutine can optionally provide a value to its parent when it yields control. This is typically used to provide feedback on the operation that the coroutine is performing.

The parent can also provide a value to the coroutine when it resumes it. This typically impacts the behavior of the coroutine. For example, this can be the result of an operation that the coroutine is waiting for or a signal to stop the coroutine.

### Intuitive Example
Consider the following [intuitive example from Idan Arye](https://dev.to/thibmaek/explain-coroutines-like-im-five-2d9). You're watching a cartoon...
- You start watching the cartoon, but it's the intro.
- Instead of watching the intro you switch to the game and enter the online lobby - but it needs 3 players and only you and your sister are in it.
- Instead of waiting for another player to join you switch to your homework, and answer the first question.
- The second question has a link to a YouTube video you need to watch. You open it - and it starts loading.
- Instead of waiting for it to load, you switch back to the cartoon. The intro is over, so you can watch.
- Now there are commercials - but meanwhile a third player has joined so you switch to the game
- And so on...

Thus, each time you are waiting for something to happen, you switch to something else. Thus, you reduce the time you spend waiting and being unproductive and maximize the time you spend doing something useful. This is the power of coroutines.

### Technical Example
Consider the following technical example. You are writing a program that inserts data into a database. The program must insert several rows every second. Each time you insert a row, you have to one second to get a response from the database.

If you implemented this program using a synchronous approach, you would have to wait for the database to respond before you could insert the next row. This would be very inefficient, as you would be spending most of your time waiting for the database to respond. Your program would not be able to insert more than one row per second.

However, if you implemented this program using a coroutine, you could insert a row into the database and then immediately switch to a different instance of the coroutine to insert another row. You can keep creating new instances of the coroutine, sending many insert requests per second. When the database responds to a request from a given instance of the coroutine, you can switch back to that instance to handle the response. This way, you can insert many rows per second, even though the database takes one second to respond to each row.

### Concurrent vs. Parallel
> [!ATTENTION]
> Coroutines allow for _concurrent_ programming, not _parallelism_. This means that coroutines are useful for writing code that switches between multiple tasks, but does not perform at the same time. For example, a coroutine can pause execution while waiting for a file to be read, and during that time, another coroutine can do something else. However, the coroutine cannot read the file and do something else at the same time.
>
> Parallelism, on the other hand, is the ability to do multiple things at the same time. This is typically achieved using multiple threads or processes, whereby multiple computations are performed in the same instant on multiple cores or processors. Parallelism is _not_ used in the task planning system for the robot.
>
> See [this](https://go.dev/blog/waza-talk) for a more in-depth explanation of the difference between concurrency and parallelism.

This also means that any program written using coroutines can, in theory, be written using regular synchronous code, and would behave identically. However, using coroutines can make the program far easier to write and maintain.

In the intuitive example above, you _never_ perform multiple tasks at the same time. You only ever perform one task at a time. However, you switch between tasks very quickly, so it _seems_ like you are doing multiple things at once. You don't waste any time waiting for something to happen. This is the essence of concurrency.

In the technical example above, you could create a new thread for each insert request – using parallelism – and achieve the same result. However, if we're inserting thousands of rows per second, this would create thousands of threads, which would be very inefficient as threads come with overhead. Using coroutines, however, we incur far less overhead.

## Generators

The history of coroutines in Python is a bit messy. There was not a single moment when coroutine were introduced, but rather a series of incremental changes to the language that together made coroutines possible.

The first step towards coroutines was the introduction of generators in Python 2.2. Generators are a special kind of function that can pause and resume their execution.

Generators are distinguished from traditional functions _only_ by their use of `yield` in the function body. `yield` is used to pause the function and return a value to the caller. When the function is resumed, it continues from where it left off.

Consider the following example of a traditional functions that returns the first `n` Fibonacci numbers:

```python
def fibonacci_list(n):
    a, b = 1, 1
    result = []
    for _ in range(n):
        result.append(a)
        a, b = b, a + b
    return result
```

We can get the first 7 Fibonacci numbers using this function as follows:

```python
>>> fibonacci_list(7)
[1, 1, 2, 3, 5, 8, 13]
```

We can iterate over the first 7 Fibonacci numbers using this function as follows:

```python
>>> for number in fibonacci_list(7):
...     print(number)
...
1
1
2
3
5
8
13
```

Now consider the following example of a generator that returns the first `n` Fibonacci numbers:

```python
def fibonacci_generator(n):
    a, b = 1, 1
    for _ in range(n):
        yield a
        a, b = b, a + b
```

If we call this function, we get a generator object:

```python
>>> fibonacci_generator(7)
<generator object fibonacci_generator at 0x7f8e3c3e3d60>
```

A generator is not a list. It is an object that can be iterated over using the built-in `next` function to produce a sequence of values. It is _not_ a sequence of values itself.

Each time you call `next` on a generator, it resumes the generator from where it left off and returns the next value that the generator yields. When the generator is exhausted, it raises a `StopIteration` exception.

```python
>>> generator = fibonacci_generator(7)
>>> next(generator)
1
>>> next(generator)
1
>>> next(generator)
2
>>> next(generator)
3
>>> next(generator)
5
>>> next(generator)
8
>>> next(generator)
13
>>> next(generator)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
```

We can iterate over the first 7 Fibonacci numbers using this generator as follows:

```python
>>> for number in fibonacci_generator(7):
...     print(number)
...
1
1
2
3
5
8
13
```
Iterating over a generator through a `for` loop is equivalent to calling `next` on the generator until it is exhausted. This does not raise a `StopIteration` exception, as the `for` loop handles it internally.

Syntax-wise, iterating over a generator through a `for` loop is identical to iterating over a list. However, the generator is _far_ more memory efficient than the list. This is because the generator does not store all of the values it yields in memory. Instead, it only stores the state of the generator. In this case the state is the values of `a` and `b` and the number of iterations left.

This makes generators ideal for working with large sequences of values where you only need to access one value at a time. For example, if you need to access the first Fibonacci number, then the second, then the third, and so on, you should use a generator.

On the other hand, if you need to access multiple values at once, you should use a list. For example, if you need to access the first 7 Fibonacci numbers repeatedly in a random order, you should use a list.

A more realistic example of a generator is when reading a file. A traditional function that reads a in its entirety a file and returns a large string would consume a lot of memory. However, if we only need to access the file one line at a time, we can use a generator that yields each successive line. This is far more memory efficient.

Generators can also be used to implement infinite sequences. For example, the following generator yields an infinite sequence of Fibonacci numbers:

```python
def fibonacci_infinite():
    a, b = 1, 1
    while True:
        yield a
        a, b = b, a + b
```

We can call the `next` function on this generator as many times as we want, and it will keep yielding Fibonacci numbers forever.

Generators' ability to pause and resume their execution makes them a possible way to implement coroutines. However, they don't provide a way to send values to the generator when it is resumed, and they don't provide a way for the generator to return a value to the caller when it yields control... _yet_.

## Generator-Based Coroutines

The next step towards coroutines was the introduction of the `send` method to generators in Python 2.5. This method allows you to send a value to a generator when it is resumed.

Consider the following example of a generator that yields the sum of the values it receives:

```python
def sum_generator():
    total = 0
    while True:
        value = yield total
        total += value
```

We can call the `send` method on this generator to send it values. Each time we call `send`, the generator resumes from where it left off. The `yield` expression resolves to the value that was sent to the generator, and the generator continues executing until it reaches the next `yield` expression where it yields the sum of the values it has received so far.

```python
>>> generator = sum_generator()
>>> next(generator)
0
>>> generator.send(1)
1
>>> generator.send(2)
3
>>> generator.send(3)
6
```

Calling `next` on the generator is equivalent to calling `send(None)`.

> [!NOTE]
> The first value sent into a generator _must_ be `None`. This is because the generator must begin execution and `yield` before it can receive a non-`None` value.

The addition of the `send` method to generators made it possible to implement coroutines using generators. Here's a generator-based implementation of the [technical example](#technical-example) from earlier:

```python
def insert_into_database(row):
    db = get_database_connection()
    db.insert(row)
    response = db.get_response()

    # Yield the response to the parent while waiting for the database to respond
    while True:
        yield response
        response = db.get_response()
        if response.status != "pending":
            break

    if response.status == "success":
        # The database has responded with a success status
        print("success")
    else:
        # The database has responded with a failure status
        print("failure")

    db.close()

rows = [row1, row2, row3, ...]
coroutines = [insert_into_database(row) for row in rows]
responses = [next(coroutine) for coroutine in coroutines]
completed_coroutines = 0

while completed_coroutines < len(coroutines):
    for i, coroutine in enumerate(coroutines):
        if responses[i]:
            try:
                responses[i] = coroutine.send(None)
            except StopIteration:
                responses[i] = None
                completed_coroutines += 1
```

This code creates a coroutine for each row that needs to be inserted into the database. It then calls `next` on each coroutine to start them, whereby each coroutine sends the insert request to the database. It then enters a loop that continues until all of the coroutines have finished. Each time through the loop, it calls `send` on each coroutine to resume it. If the coroutine is still waiting for the database to respond, it yields the pending response to the parent. If the database has responded, the coroutine processes the response and closes the database connection, resulting in a StopIteration exception. The exception is caught and the number of completed coroutines is incremented. The loop continues until all of the coroutines have finished.

Even though we could now create coroutines in Python, there was no easy way to nest coroutines, and there was no way to return a final value from a coroutine when it was exhausted... _yet_.

### `yield from` and `return`

Python 3.3 introduced the `yield from` expression, which made it possible to delegate the execution of a generator to another generator. This made it possible to nest coroutines in a more natural way.

If a parent coroutine delegates the execution of a child coroutine to another coroutine using the `yield from` expression, the parent coroutine will pause and resume whenever the child coroutine pauses and resumes. Any values that the child coroutine yields will be passed to the parent's caller, and any values that the parent's caller sends to the parent will be passed to the child coroutine. When the child coroutine is exhausted, the parent coroutine will resume execution after the `yield from` expression. The StopIteration exception will _not_ be raised in the parent; it will be caught automatically by the `yield from` expression.

Python 3.3 also introduced the `return` statement to generators, which made it possible to return a final value from a coroutine when it was exhausted.

Consider the following example:

```python
def insert_into_database(row):
    db = get_database_connection()
    ... # Same as previous
    db.close()
    return response

def select_from_database(query):
    db = get_database_connection()
    db.select(query)
    response = db.get_response()

    # Yield the response to the parent while waiting for the database to respond
    while True:
        yield response
        response = db.get_response()
        if response.status != "pending":
            break

    if response.status == "success":
        # The database has responded with a success status
        print("success")
    else:
        # The database has responded with a failure status
        print("failure")

    db.close()
    return response

def insert_and_select_from_database(row, query):
    insert_response = yield from insert_into_database(row)
    select_response = yield from select_from_database(query)
    return insert_response, select_response
```

Here, `insert_and_select_from_database` is the parent coroutine that inserts a row into the database and performs a select query. It does this by delegating the insertion to `insert_into_database` and selection to `select_from_database`, using the `yield from` expression. When the child coroutines are exahusted, they return their final responses to the parent coroutine, which returns both.

The event loop that drives the program performing insertions and selects can be written in almost exactly the same way as the event loop that drives the program performing insertions only. This is possible because yielded values automatically propagate up to the event loop, and sent values automatically propagate down to `insert_into_database` and `select_from_database`; they don't get stuck in `insert_and_select_from_database`.

```python
rows = [row1, row2, row3, ...]
queries = [query1, query2, query3, ...]
coroutines = [insert_and_select_from_database(row, query)
                for row, query in zip(rows, queries)]
... # Same as previous
```

While generators now offer all the features needed to implement coroutines, the syntax for using them is a bit messy. There's no way to tell if a function is a coroutine just by looking at its definition, and `yield from` combines two keywords into one expression, which can be confusing, ... _yet_.

## `async` and `await`

Python 3.5 introduced the `async` and `await` keywords, which made it possible to write coroutines using much cleaner syntax and made coroutines first-class citizens in the language.

The `async` keyword is placed before `def` to define a coroutine, and the `await` keyword is used to delegate execution to another coroutine or Awaitable object.

> [!NOTE]
> Now, when we say "coroutine", we are referring to a function defined with `async def`, _not_ generator-based coroutines. When an `async` function is called, it returns a coroutine object, _not_ a generator object.
>
> In other words, even though it is still possible to define coroutines using generators, _native_ Python coroutines are defined using `async def`.

An Awaitable object is an object that can be used in an `await` expression. It can be a coroutine or an object that implements the `__await__` method. The `__await__` method must return an iterator.

> [!ATTENTION]
> If you use `yield` inside a function defined with `async`, the function is **not** a coroutine. It is an _asynchronous generator_. This is a different concept that is not covered in this article and is not used in the task planning system for the robot.
>
> To pause a coroutine and provide a value to the parent, you must `await` an Awaitable object that uses `yield`.

