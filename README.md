# Async-PyO3-Examples
A repo to help people to interact with AsyncIO with PyO3 native extensions.

## A Word Of Warning
Please read through these warnings before jumping into this, asyncio at the low level is fairly complicated and is definitley **not** recommended for begginers who have little to no understanding of AsyncIO.

#### Writing coroutines in Rust and Pyo3 will not make them 'faster'
creating and running the coroutine will actually be *slower* on average so please do not enter this believing that if you write all your coroutines in Rust just to gain execution speed, They'll only be faster if what you plan todo inside them can be sped up by Rust but *need* to be coroutines in order to achieve that.

#### Low level coroutines are *not* simple 
No matter how you look at it coroutines in PyO3 will not be easy to write, you have no syntactic sugar methods like `async` and `await` (The Rust future implementations do not work).
You will also be without yeild from (under the hood `await`) so you should think of how you are going to structure your code going into it.


## How AsyncIO works in Python under the hood
Python's AsyncIO is built off the idea of a event loop and generator coroutines allowing functions to be suspended and resumed which is managed by the event loop when a `await` block is hit. This gives us the affect of single threaded concurrency.

### Breaking down async/await
Since Python 3.5.5 synatic sugar methods of `async` and `await` make our lives alot easier by doing all the heavy lifting for us keeping all the messy internals hidden away.
However, this wasn't always a thing. In Python versions older than 3.5.5 we were left with manually decorating our functions and using `yield from` (more on this later), if you've worked with async in Python 2 or old versions of Python 3 this might come easier to you.

#### What is `await` under the hood?
await is basically a `yield from` wrapper after calling the coroutine and then calling `__await__`, which is all we really need to know for the sake of writing async extensions in Rust, the next question you will probably have is "well... What's yield from?", glad you asked! All we need to know about yield from is that we yield the results from another iterator until it's exhuased (All coroutines are generators so all support being iterated over which will support yield from) however, we cannot use `yield from` in our Rust code so here's what we would do logically in Rust (Implemented in Python).

**Example code:**
```py
# python 3.8

# We defined our coroutine using the normal async system for now,
# we should see "being called" and then "hello" being returned by utilising
# yield from.
async def foo():    
    print("foo is being called")
    return "hello"

# We can still call our coroutine like normal
my_iterator = foo()

# We have to call __await__ as part of the async API
my_iterator = my_iterator.__await__()

# We have to call __iter__ to expose the 
# generator aspect of the coroutine.
my_iterator = my_iterator.__iter__()

# Because of the generator nature of coroutines,
# we expect it to raise a StopIteration error which
# gives us our resulting value.
try:
    while True:             # Keep going till it errors
        next(my_iterator)
except StopIteration as result:
    print(f"Result of our coroutine: {repr(result.value)}")
```

**Output:**
```
>>> foo is being called
>>> Result of our coroutine: 'hello'
```

#### What is `async def` under the hood?
`async def` or `async` key words are alot more complicated under the hood than `await`.
Before `async` became a keyword you would have to decorate your function with `@asyncio.coroutine` to wrap your code in a coroutine class, however we have to re-create the coroutine class itself.

**A coroutine remake in Python**
```py
# python 3.8
import asyncio


# We're going to try recreate this coroutine by
# making a class with the required dunder methods.
#
# This coroutine just returns the parameter it's given.
async def my_coroutine(arg1):
    return arg1


# Our coroutine copy takes arg1 just like my_coroutine,
# we can save this to somewhere, in this case we're setting it
# to arg1 as a instance variable.
class MyCoroutineCopy:
    def __init__(self, arg1):
        self.arg1 = arg1

    # __await__ is required to register with the `await` keyword returning self.
    def __await__(self):
        return self

    # __iter__ is used just to return a iterator, we dont need this to be self
    # but for the sake of this we're using the class itself as a iterator.
    def __iter__(self):
        return self

    # __next__ is used to make our coroutine class a generator which can either
    # return to symbolise a yield or raise StopIteration(value) to return the
    # output of the coroutine
    def __next__(self):
        # we'll do what my_coroutine does and echo the first parameter
        raise StopIteration(self.arg1)


async def main():
    result = await my_coroutine("foo")
    print(f"my_coroutine returned with: {repr(result)}")

    result = await MyCoroutineCopy("foo")
    print(f"MyCoroutineCopy returned with: {repr(result)}")


asyncio.run(main())
```

**Output:**
```
my_coroutine returned with: 'foo'
MyCoroutineCopy returned with: 'foo'
```

## Converting It To Rust
Now we've got all the concepts out of the way and under our tool belt we can actually start to recreate this is Rust using PyO3.

We'll start by breaking down and recreating our first Python example recreating a `await`:

**Some Boilerplate:**
```rust
// lets get our basics setup first
use pyo3::prelude::*;

// We're going to create a basic function just to house our awaiting
#[pyfunction]
fn await_foo(py: Python, foo: PyObject) {
    // todo
}
 ```


