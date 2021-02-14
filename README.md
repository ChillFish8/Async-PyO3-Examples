# Async-PyO3-Examples
A repo to help people to interact with AsyncIO with PyO3 native extensions.

# Contents

- **[Introduction](https://github.com/ChillFish8/Async-PyO3-Examples/blob/main/README.md#introduction---a-word-of-warning)** *If you're new to this book you will want to read this*
- **[Breaking Down Python](https://github.com/ChillFish8/Async-PyO3-Examples/blob/main/README.md#breaking-down-python---how-asyncio-works-in-python-under-the-hood)** *Goes through the internals of how AsyncIO with async and await is implemented*
- **[Implementing Rust](https://github.com/ChillFish8/Async-PyO3-Examples/blob/main/README.md#implementing-rust---in-progess)** *Takes what we learnt from [Breaking Down Python]() and applies it to our Rust code* - **In Development**


## Introduction - A Word Of Warning
Please read through these warnings before jumping into this, asyncio at the low level is fairly complicated and is definitley **not** recommended for begginers who have little to no understanding of AsyncIO.

#### Writing coroutines in Rust and Pyo3 will not make them 'faster'
creating and running the coroutine will actually be *slower* on average so please do not enter this believing that if you write all your coroutines in Rust just to gain execution speed, They'll only be faster if what you plan todo inside them can be sped up by Rust but *need* to be coroutines in order to achieve that.

#### Low level coroutines are *not* simple 
No matter how you look at it coroutines in PyO3 will not be easy to write, you have no syntactic sugar methods like `async` and `await` (The Rust future implementations do not work).
You will also be without yeild from (under the hood `await`) so you should think of how you are going to structure your code going into it.


## Breaking Down Python - How AsyncIO works in Python under the hood
Python's AsyncIO is built off the idea of an event loop and generator coroutines allowing functions to be suspended and resumed which is managed by the event loop when a `await` block is hit. This gives us the effect of single threaded concurrency.

### Breaking down async/await
Since Python 3.5.5 the syntactic sugar keywords `async` and `await` make our lives a lot easier by doing all the heavy lifting for us, keeping all the messy internals hidden away.
However, this wasn't always a thing. In Python versions older than 3.5.5 we were left with manually decorating our functions and using `yield from` (more on this later), if you've worked with async in Python 2 or old versions of Python 3 this might come easier to you.

#### What is `await` under the hood?
await is basically a `yield from` wrapper after calling the coroutine and then calling `__await__`, which is all we really need to know for the sake of writing async extensions in Rust, the next question you will probably have is "well... What's yield from?", glad you asked! All we need to know about yield from is that we yield the results from another iterator until it's exhausted (All coroutines are generators so all support being iterated over which will support yield from) however, we cannot use `yield from` in our Rust code so here's what we would do logically in Rust (Implemented in Python).

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
`async def` or `async` key words are a lot more complicated under the hood than `await`.
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
    # but for the sake of this we're using the class itself as an iterator.
    # Note that it is required for old-style generator-based coroutines compatibility.
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

---

### A look behind the scenes

So far, we've only showed you how to make a coroutine, not how to make a coroutine *asynchronous*.
That is, how to interact with *other* systems in a *non-blocking* way (disk access, network access, other threads, etc. Everything that we call I/O.). All the code that we've shown so far, even if using `async` and `await` keywords, is synchronous, it runs sequentially.

Here's the proof:

``` python
# python 3.8
import asyncio


async def level3(i):
    print(f"task {i} reached the bottom, exiting...")


async def level2(i):
    print(f"task {i} entering level 3")
    result = await level3(i)
    print(f"task {i} exiting level 3")


async def level1(i):
    print(f"task {i} entering level 2")
    result = await level2(i)
    print(f"task {i} exiting level 2")


async def main():
    return await asyncio.gather(*[level1(i) for i in range(3)])


asyncio.run(main())
```

The above code will always, deterministicly, produce the following output. (Try and play with the code a little, to convince yourself.)

```
task 0 entering level 2
task 0 entering level 3
task 0 reached the bottom, exiting...
task 0 exiting level 3
task 0 exiting level 2
task 1 entering level 2
task 1 entering level 3
task 1 reached the bottom, exiting...
task 1 exiting level 3
task 1 exiting level 2
task 2 entering level 2
task 2 entering level 3
task 2 reached the bottom, exiting...
task 2 exiting level 3
task 2 exiting level 2
```

And this is good! As long as there is nothing to wait for, we just chain the iterators, and keep going down the async stack, and your CPU doesn't waste cycles on expensive context switches.

So, to sum it up, `async` and `await` are the building blocks of asynchronous computations, they *enable* asynchronicity, but they don't, *by themselves*, make your program magically asynchronous.

For that, we're missing a key element that lies in deep in the guts of asyncio, because just like with any magic, there's a trick, and to find it, we have to take a look behind the scenes.

That missing piece of the puzzle is - well - the `Future`.
Not the Rust one, the [`asyncio.Future`](https://docs.python.org/3.8/library/asyncio-future.html#asyncio.Future).

In essence, if you think of the async call stack as a tree, the Futures are always the leaves.
They represent the very moment where you are about to wait for an I/O operation, and you want to tell the async loop "Alright, I can't go any further now, could you please put my parent task on hold?
**Someone will *call* you *back* when I'm ready to proceed**"

#### Implementing your own (Python) future

So, to illustrate, let's implement our very own dumb future, that will spawn a thread to wake it up after some time.

``` python
# python 3.8
import asyncio
from asyncio.exceptions import InvalidStateError
import threading
import time


class MyFuture:
    def __init__(self, result):
        """ So here is our future. From the user's perspective, it takes `result` in
            and will spit the square of that result after 1s when you await it.
        """
        self._result = result
        # The loop is here for asyncio's sanity checks but also to schedule the
        # resuming of the task once the future is completed
        self._loop = None
        # The callbacks is a list of functions to be called when the future completes
        self._callbacks = []
        # Now this is an ugly bit as it serves a double purpose:
        # First, the loop will check that it's present as a flag of whether this
        # future is advertising itself to be asyncio compatible or not.
        # Second, it's value determines if the Task should be put on the waiting
        # queue. This is exactly what we want, so we set it to True.
        self._asyncio_future_blocking = True

    def __await__(self):
        # Now that we're in an async context, we store the loop
        self._loop = asyncio.get_running_loop()


        def sleep_and_return(result):
            """ Here is what's going to run in our thread.
                We simulate some computations, and then return the result.
                The task will be woken up on the next cycle of the loop.
                Note that we're using the thread-safe variant of call_soon.
                (This is all to avoid deadlocks.)
            """
            time.sleep(1)
            # More about `self._set_result` later
            self._loop.call_soon_threadsafe(self._set_result, result * result)


        threading.Thread(target=sleep_and_return, args=(self._result,)).start()
        # We forget our result. It will be the thread's job to set it back.
        self._result = None
        # We are the iterator
        return self

    def __next__(self):
        # This will be called immediately afterwards, but the thread won't be
        # finished yet, so...
        if self._result is None:
            # ... Here's the trick! We yield... ourself! A future!
            return self
            # Asyncio will now check _asyncio_future_blocking... True.
            # "So, we have a future here, let's check its loop... get_loop()?"
        raise StopIteration(self._result)

    def get_loop(self):
        return self._loop
        # "... Yes, the loop matches, alright, let's give it a callback..."

    def add_done_callback(self, callback, *, context=None):
        self._callbacks.append((callback, context))
        # "... Put it on the waiting queue, and forget about it."
        # Now the loop is free to do something else, or sleep...
        # Until our thread finishes and calls:
    
    def _set_result(self, result):
        """ This is our waker. It will set the result and schedule the
            callbacks to resume our task.
            One very important note is that it's a separate function as it
            needs to be an atomic operation that happens on the main thread.
            (Otherwise the loop might resume before the external thread is
            finished and we end up in a deadlock.)
        """
        self._result = result
        # This code is taken straight from asyncio's source code
        for cb, ctx in self._callbacks:
            self._loop.call_soon(cb, self, context=ctx)
        self._callbacks[:] = []

    # Now, before resuming, the first thing asyncio does is call `.result()`
    # Don't ask me why, as the return is just ignored, but it does.
    # Maybe it's there to mirror `concurrent.futures.Future`, and the source
    # implementation uses it in the `__await__` function so maybe that's to avoid
    # code duplication ? But then why call it as part of the loop and force it
    # as part of the Future API? I just don't know
    def result(self):
        if self._result is None:
            # This is never called, the exception is there to prove you that the
            # task will only be resumed once the result is ready once again
            raise InvalidStateError('result is not ready')
        return None
        # Now the task is resumed and __next__ is called a final time to
        # provide the result.

async def wrapper(i):
    """ Simulating some depth in the call async stack.
        (It also lets us make the shortcut of initialising _asyncio_future_blocking
        to True.)
    """
    return await MyFuture(i)

async def main():
    """ To show you that this is indeed async, let's spawn a thousand of those
        futures that will all complete after 1 second.
    """
    results = await asyncio.gather(*[wrapper(i) for i in range(1000)])
    print(results)

asyncio.run(main())
```

---

## Implementing Rust
Now we've got all the concepts out of the way and under our tool belt we can actually start to recreate this in Rust using PyO3.

We'll start by breaking down and recreating our first Python example recreating a `await`:
<br><br>
### Re-Creating Python's `await` in Rust
**Setting up boilerplate:**
This is just our standard bit of setup, if you already have an existing bit of code with this in, you can ignore it.
Though for the purposes of learning it would be a good idea to have a seperate area to mess with before putting it in your actual code.

```rust
// lets get our basics setup first
use pyo3::prelude::*;
use pyo3::wrap_pyfunction;


// We're going to create a basic function just to house our awaiting
#[pyfunction]
fn await_foo(py: Python, foo: PyObject) -> PyResult<()> {
    // We'll add our code here soon
}


// Lets just call our module await_from_rust for simplicity.
#[pymodule]
fn await_from_rust(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(await_foo, m)?).unwrap();
    Ok(())
}
```

**Producing our iterator from a coroutine:**
A few things differ when we want to setup our coroutine call in Rust because we can make use of the C api directly which can help with efficency. 
`call_method`'s are generally quite expensive calls so reducing them where we can has the potential to improve our performance.

Our code to produce the iterator, notice how we use `iter()` instead of `call_method0(py, "__iter__")`, 
always a good idea to check the docs to see if the thing you're doing supports direct calls rather than going through `call_method`.
```rust
// We call it like we did in python
let mut my_iterator  = foo.call0(py)?;

// We call __await__ on the coroutine using call_method0 for no parameters
// as part of the async API
my_iterator = my_iterator.call_method0(py, "__await__")?;

// We call __iter__  on the coroutine using call_method0 for no parameters
// to expose the generator aspect of the coroutine
my_iterator = my_iterator.iter()?;
```

**actually using our iterator**
Now we have our iterator we can use `loop` to call next until it's exhuasted. Unlike Python we dont have try/except (not in the same way atleast) so our loop is positioned slightly diffrent than our Python example.

```rust
// Unlike python we cant wrap this in a try except / we dont want to.
// so for this we'll just use loop to iterate over it like While True
// and match the result.
let mut result: PyResult<PyObject>;     // Saves us constantly re-declaring it.
loop {

    // We call __next__ which is all the next() function does in Python.
    result = my_iterator.call_method0(py, "__next__");

    // lets match if the iterator has returned or raised StopIteration.
    // For this example we are assuming that no other error will ever be
    // raised for the sake of simplicity.
    match result {
        // If its okay (it returned normally) we call next again.
        Ok(r) => continue,

        // if it errors we know we're done.
        Err(e) => {
            if let Ok(stop_iteration) = e.pvalue(py).downcast::<PyStopIteration>() {
                let returned_value = stop_iteration.getattr("value")?;
                    
                // Let's display that result of ours
                println!("Result of our coroutine: {:?}", returned_value);

                // Time to escape from this while loop.
                return Ok(());
             } else {
                return Err(e);
             } 
                
        }
   };
}
```


**our finished function that `awaits` a coroutine:**
```rust
// lets get our basics setup first
use pyo3::prelude::*;
use pyo3::wrap_pyfunction;


// We're going to create a basic function just to house our awaiting
#[pyfunction]
fn await_foo(py: Python, foo: PyObject) -> PyResult<()> {

    // We call it like we did in python
    let mut my_iterator  = foo.call0(py)?;

    // We call __await__ on the coroutine using call_method0 for no parameters
    // as part of the async API
    my_iterator = my_iterator.call_method0(py, "__await__")?;

    // We call __iter__  on the coroutine using call_method0 for no parameters
    // to expose the generator aspect of the coroutine
    my_iterator = my_iterator.iter()?;

    // Unlike python we cant wrap this in a try except / we dont want to.
    // so for this we'll just use loop to iterate over it like While True
    // and match the result.
    let mut result: PyResult<PyObject>;     // Saves us constantly re-declaring it.
    loop {

        // We call __next__ which is all the next() function does in Python.
        result = my_iterator.next();

        // lets match if the iterator has returned or raised StopIteration.
        // For this example we are assuming that no other error will ever be
        // raised for the sake of simplicity.
        match result {
            // If its okay (it returned normally) we call next again.
            Ok(r) => continue,

            // if it errors we know we're done.
            Err(e) => {
                if let Ok(stop_iteration) = e.pvalue(py).downcast::<PyStopIteration>() {
                    let returned_value = stop_iteration.getattr("value")?;
                    
                    // Let's display that result of ours
                    println!("Result of our coroutine: {:?}", returned_value);

                    // Time to escape from this while loop.
                    return Ok(());
                } else {
                    return Err(e);
                } 
                
            }
        };
    }

    // Rust will warn that this is unreachable, probably a good idea to
    // add a timeout in your actual code.
    Ok(())
}

#[pymodule]
fn await_from_rust(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(await_foo, m)?).unwrap();
    Ok(())
}
```
<br>

### Re-Creating Python's `async def`/coroutines in Rust
Great! Now we've managed to `await` our coroutines and understand that concept we can start to build our own coroutines.<br>
Now this is where it starts to get a little bit more complicated because we are without `yield` or `yield from` directly, 
however, good news! PyO3 already implements some helper functions to make our life a little bit easier with a `yield` function and `return` function for iterators.
Other than that we need to implement the requrires dunder methods (double underscore methods e.g `__init__`) this cannot be done via simply making a function called it unless specificially stated.
<br><br>

We're going to use the `#[pyproto]` macro for a couple things:
- For implementing `__await__` by implying the `PyAsyncProtocol`
- For implementing the `__iter__` and `__next__` methods by implying the `PyIterProtocol`, see [the docs for more about iterators in PyO3](https://pyo3.rs/v0.12.3/class/protocols.html?highlight=Iter#iterator-types)
<br><br>

**Setting up our boilerplate:**<br>
Like we did with `await` we need to setup some basic boiler plate for the sake of demonstration.

Our struct `MyAwaitable` will form the basis for our awaitable, it is important to note now that this will not be identified as a `coroutine` type by Python but as an awaitable type instead.
This will mean things like `asyncio.create_task()` wont work but `asyncio.ensure_future()` will.

```rust
// lets get our basics setup first
use pyo3::prelude::*;

// Our base Python class that will do the job of Python's
// coroutine class.
#[pyclass]
struct MyAwaitable {} 

// Let's make it an other module since we're now doing the opposite: awaitable_rust
#[pymodule]
fn awaitable_rust(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_class::<MyAwaitable>()?;
    Ok(())
}
```

**Making it 'awaitable'**<br>
Python has a very simple system for making an object awaitable, `await` simply  calls `__await__` under the hood. We can recreate this using the `pyproto` macro and the `PyAsyncProtocol`.

```rust
// lets get our basics setup first
use pyo3::prelude::*;

// Our base Python class that will do the job of Python's
// coroutine class.
#[pyclass]
struct MyAwaitable {} 

// Adding out async protocol, this makes us awaitable.
// It should be important to note: DO NOT LISTEN TO YOUR LINTER.
// Your linter can get very, very confused at this system and will
// want you to implement things that you do *not* want to implement
// to 'satisfy' this protocol implementation.
// Even if it highlights in red the below implementation will stil compile.
#[pyproto]
impl PyAsyncProtocol for MyAwaitable  {   
    fn __await__(slf: PyRef<Self>) -> PyRef<Self> {
        slf     // We're saying that we are the iterable part of the coroutine.
    }
}

// let's make it an other module since we're now doing the opposite: awaitable_rust
#[pymodule]
fn awaitable_rust(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_class::<MyAwaitable>()?;
    Ok(())
}
```

*Wow! is it that easy to make a coroutine?* - Sadly not quite, the `slf` reference does not allow us to internally call functions we've defined, in the above example as well we are telling Python that our iterable is itself. This will crash if we try to run this on Python now because we're missing the iterator protocol.

However, this simple setup still carries a lot of use. If you have something that just needs to be awaitable and transfer some pre-computed fields to an existing awaitable or PyObject you can just create the object -> call `__await__` and return that. This can make things considerably easier if your Rust coroutines are simply acting as a middle man for some efficent code.

**Making our awaitable an iterable**<br>
It should be important to note that just because something is awaitable does not make it a coroutine, coroutines are essentially self contained classes that return `self` on both `__await__` and `__iter__` calls and execute the actual code upon the `__next__` call (Please note I am heavily simplifying it to make sense of the following Rust code.)

Just like we did with `__await__` we can use `pyproto` to implement the iterable dunder methods:
```rust
// lets get our basics setup first
use pyo3::prelude::*;
use pyo3::iter::IterNextOutput;
use pyo3::PyIterProtocol;
use pyo3::PyAsyncProtocol;
use pyo3::wrap_pyfunction;

// Our base Python class that will do the job of Python's
// coroutine class.
#[pyclass]
struct MyAwaitable {} 

// Adding out async protocol, this makes us awaitable.
#[pyproto]
impl PyAsyncProtocol for MyAwaitable {   
    fn __await__(slf: PyRef<Self>) -> PyRef<Self> {
        slf     // We're saying that we are the iterable part of the coroutine.
    }
}

#[pyproto]
impl PyIterProtocol for MyAwaitable {
    // This is an optional function. If you dont want to do anything like returning
    // an existing iterator, dont worry about implementing this.
    fn __iter__(slf: PyRef<Self>) -> PyRef<Self> {
        slf
    }
    
    // There are other return types you can give, however IterNextOutput is by far the biggest
    // helper you will get when making awaitables.
    fn __next__(_slf: PyRefMut<Self>) -> IterNextOutput<Option<PyObject>, &'static str> {
        IterNextOutput::Return("Ended")
    }
}

// Exposing our custom awaitable to Python.
// This will behave like a coroutine.
#[pyfunction]
fn my_awaitable() -> MyAwaitable {
    MyAwaitable {}
}

// let's make it an other module since we're now doing the opposite: awaitable_rust
#[pymodule]
fn awaitable_rust(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_class::<MyAwaitable>()?;
    m.add_function(wrap_pyfunction!(my_coroutine, m)?).unwrap();
    Ok(())
}
```

And now we can call our Rust coroutine from Python.

```py
# python 3.8
import asyncio

import awaitable_rust


async def main():
    # Everything works as if it was coroutine...
    result = await awaitable_rust.my_awaitable()
    print(f"my_coroutine returned with: {result!r}")

# But note that this won't work:
# asyncio.run(await_from_rust.my_coroutine())
# because asyncio won't recognise it as a `coroutine` object

asyncio.run(main())
```

### Using Rust's futures as Python awaitables

What if you could turn your rust `async` functions into awaitable Python objects?

Turns out, Rust's and Python's `async`/`await` models aren't that different and can be made to work together.

#### Comparison of Python's and Rust's `async`/`await` models

[Earlier](#implementing-your-own-python-future) we broke down how futures work in python, which should hopefully have given us a good enough understanding of the Python `async`/`await` strategy.

Now, if you haven't already, you should read [this chapter](https://rust-lang.github.io/async-book/02_execution/01_chapter.html) of the async Rust book which does an excellent job at explaining how `Futures` work in Rust. 

And armed with this knowledge we should have everything we need to write the glue to make those two work together, but for the sake of this explanation, let's recap:

- In Python (asyncio), `async def`s translate to `coroutine`s.
These coroutines are just generators that yield from other generators (coroutines) whenever they encounter an `await` statement until they hit an `asyncio.Future` (a leaf) down the chain.
Then the whole `Task` (stack of generators/coroutines) is put on the waiting queue, and callbacks are setup on the Future to resume the Task once ready.
This process repeats until the Task finishes with a result.
- In Rust, async blocks are compiled down to state machines that advance between each state of waiting when polled, until they reach the final Ready state.
This is conceptually equivalent to Python's approach:
async blocks are syntactic sugar for structs that `impl` the `std::future::Future` trait by wrapping other objects that `impl` the trait.
Somewhere down the line, there will be a leaf Future that actually waits for some I/O and implements the trait manually.
Along this line is passed a `Waker` that will be used to signal when the whole (compiled) Future is ready to be resumed.

In summary, async blocks are coroutines, wakers are callbacks, futures are futures, and future stacks compiled to state machines are Tasks.

#### Implementing a Python future in Rust

To hammer down what we've learned so far and as a first step, let's reimplement [our asyncio future](#implementing-your-own-python-future) in Rust.
I recommend you keep the Python version opened on the side to follow along.

```rust
use {
    pyo3::{iter::IterNextOutput, prelude::*, types::IntoPyDict, PyAsyncProtocol, PyIterProtocol},
    std::{thread, time::Duration},
};

// So this is our class, except it's now implemented in Rust!
// I have kept the same layout so you can follow the corresponding Python code.
#[pyclass]
struct MyFuture {
    // `result` is now concretized to a u32 for the sake of the example, where
    // in Python it could have been anything. You can use PyObject if you want.
    result: Option<u32>,
    aio_loop: Option<PyObject>,
    callbacks: Vec<(PyObject, Option<PyObject>)>,
    #[pyo3(get, set)]
    _asyncio_future_blocking: bool,
}

#[pyproto]
impl PyAsyncProtocol for MyFuture {
    fn __await__(slf: Py<Self>) -> PyResult<Py<Self>> {
        // Only locking the GIL for what we need
        let (result, aio_loop) = Python::with_gil(|py| -> PyResult<_> {
            let mut slf = slf.try_borrow_mut(py)?;
            let aio_loop: PyObject = PyModule::import(py, "asyncio")?
                .call0("get_running_loop")?
                .into();
            slf.aio_loop = Some(aio_loop.clone());
            Ok((slf.result.take(), aio_loop))
        })?;
        // We know we have a result thanks to the contstructor
        let result = result.expect("result uninitialised");

        // Now this is not obvious at first, but think of our Python implementation,
        // when we passed `self._set_result` to `call_soon_threadsafe`.
        // `self._set_result` is actually an object.
        // An object that wraps (or closes over) some pointers to call a function.
        // This is our handcrafted lambda to wake our task.
        #[pyclass]
        struct WakerClosure {
            result: u32,
            aio_loop: PyObject,
            future: Py<MyFuture>,
        }

        let future = slf.clone();
        thread::spawn(move || {
            thread::sleep(Duration::from_secs(1));
            let waker = WakerClosure {
                result: result * result,
                aio_loop: aio_loop.clone(),
                future,
            };
            Python::with_gil(|py| aio_loop.call_method1(py, "call_soon_threadsafe", (waker,)))
                .expect("exception thrown by the event loop (probably closed)");
        });

        // When the thread is finished, on its next iteration, the loop will
        // call our waker, which in turn will call the registered callbacks.
        #[pymethods]
        impl WakerClosure {
            #[call]
            pub fn __call__(slf: PyRef<Self>) -> PyResult<()> {
                let py = slf.py();
                // We should be the only ones borrowing the future.
                // The only moment when that's not true, is when the future
                // finishes so fast, that it tries to wake the task before the
                // callbacks are registered. But in this example it never happens.
                let mut future = slf.future.try_borrow_mut(py)?;
                future.result = Some(slf.result);
                // The same code from asyncio, only translated to rust.
                let callbacks = std::mem::take(&mut future.callbacks);
                for (callback, context) in callbacks {
                    slf.aio_loop.call_method(
                        py,
                        "call_soon_threadsafe",
                        (callback, &future),
                        Some(vec![("context", context)].into_py_dict(py)),
                    )?;
                }
                Ok(())
            }
        }

        // Again, why we need to do this 2-step callback here, is because we 
        // absolutely cannot let the task be woken up before the thread finishes.
        // Otherwise we end up in a deadlock, with the main thread waiting for 
        // our child thread, which waits for the main thread to release the GIL.
        // So we avoid shared state in the child thread, and keep its 
        // interaction with the GIL to a bare minimum.
        // Now, in this simple example it might seem a bit contrived, as we have
        // control over everything the thread does, but it will become clear
        // why it's necessary when Rust futures are involved.

        Ok(slf)
    }
}

// The rest is a pretty straight-forward translation.

#[pyproto]
impl PyIterProtocol for MyFuture {
    fn __next__(slf: PyRef<Self>) -> IterNextOutput<PyRef<Self>, u32> {
        match slf.result {
            None => IterNextOutput::Yield(slf),
            Some(result) => IterNextOutput::Return(result),
        }
    }
}

#[pymethods]
impl MyFuture {
    #[new]
    fn new(result: u32) -> Self {
        Self {
            result: Some(result),
            aio_loop: None,
            callbacks: vec![],
            _asyncio_future_blocking: true,
        }
    }

    fn get_loop(&self) -> Option<&PyObject> {
        self.aio_loop.as_ref()
    }

    fn add_done_callback(&mut self, callback: PyObject, context: Option<PyObject>) {
        self.callbacks.push((callback, context));
    }

    fn result(&self) -> Option<PyObject> {
        None
    }
}

#[pymodule]
fn awaitable_rust(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_class::<MyFuture>()?;
    Ok(())
}
```

And of course our little shim of Python to see it in action.

```python
# python 3.8
import asyncio
import time
# Bonus! You can try it with a different asyncio-compatible loop
#import uvloop

import awaitable_rust

async def wrapper(i):
    return await awaitable_rust.MyFuture(i)

async def main():
    results = await asyncio.gather(*[wrapper(i) for i in range(1000)])
    print(results)


#uvloop.install()
asyncio.run(main())

```

#### Bridging the gap

For the next step, let's throw some *Rust* futures in the mix.
We will turn the thread code into a struct implementing the `std::future::Future` trait and make our *Python* future await it.

For that we will simply be adapting [the `TimerFuture` from the Asynchronous Programming in Rust book](https://rust-lang.github.io/async-book/02_execution/03_wakeups.html#applied-build-a-timer) as it (not so) coincidentally looks very much like what we have been implementing so far.

```rust
use {
    futures::{
        future::{BoxFuture, FutureExt},
        task::{waker_ref, ArcWake},
    },
    pyo3::{
        iter::IterNextOutput, prelude::*, types::IntoPyDict, wrap_pyfunction, PyAsyncProtocol,
        PyIterProtocol,
    },
    std::{
        future::Future,
        pin::Pin,
        sync::{Arc, Mutex},
        task::{Context, Poll, Waker},
        thread,
        time::Duration,
    },
};

// Straight from Asynchronous Programming in Rust, I have removed the comments
// and commented what I changed.
pub struct TimerFuture {
    shared_state: Arc<Mutex<SharedState>>,
}

struct SharedState {
    /// This bool is now an option with our result
    result: Option<u32>,

    waker: Option<Waker>,
}

impl Future for TimerFuture {
    type Output = u32; // The Future now returns something
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let mut shared_state = self.shared_state.lock().unwrap();
        match shared_state.result {
            Some(result) => Poll::Ready(result),
            None => {
                shared_state.waker = Some(cx.waker().clone());
                Poll::Pending
            }
        }
    }
}

impl TimerFuture {
    /// The timer is now hardcoded to 1 and we take the result to return instead
    pub fn new(result: u32) -> Self {
        let shared_state = Arc::new(Mutex::new(SharedState {
            result: None,
            waker: None,
        }));

        // And here is the gist of it. We're essentially still doing the same thing:
        // Spawn a thread, wait 1s, square the result, set the result, wake the task
        let thread_shared_state = shared_state.clone();
        thread::spawn(move || {
            thread::sleep(Duration::from_secs(1));
            let mut shared_state = thread_shared_state.lock().unwrap();
            shared_state.result = Some(result * result);
            if let Some(waker) = shared_state.waker.take() {
                // Note here that we are still locking the shared state when
                // calling the waker. This will be relevant later.
                waker.wake()
            }
        });

        TimerFuture { shared_state }
    }
}

#[pyclass]
struct MyFuture {
    // Now we're encapsulating a Rust future
    future: BoxFuture<'static, u32>,
    aio_loop: Option<PyObject>,
    callbacks: Vec<(PyObject, Option<PyObject>)>,
    #[pyo3(get, set)]
    _asyncio_future_blocking: bool,
    // And we keep a reference to the waker that we will be passing to the
    // rust future
    waker: Option<Arc<WakerClosure>>,
}

impl MyFuture {
    // The constructor is now internal and takes a Future that must be Send + 'static
    // in order to be `.boxed()` (Python couldn't possibly pass us a Future anyway)
    fn new(future: impl Future<Output = u32> + Send + 'static) -> Self {
        Self {
            future: future.boxed(),
            aio_loop: None,
            callbacks: vec![],
            _asyncio_future_blocking: true,
            waker: None,
        }
    }
}

#[pyproto]
impl PyAsyncProtocol for MyFuture {
    fn __await__(slf: Py<Self>) -> PyResult<Py<Self>> {
        let py_future = slf.clone();
        Python::with_gil(|py| -> PyResult<_> {
            let mut slf = slf.try_borrow_mut(py)?;
            let aio_loop: PyObject = PyModule::import(py, "asyncio")?
                .call0("get_running_loop")?
                .into();
            slf.aio_loop = Some(aio_loop.clone());
            slf.waker = Some(Arc::new(WakerClosure {
                aio_loop,
                py_future,
            }));
            Ok(())
        })?;

        Ok(slf)
    }
}

#[pyproto]
impl PyIterProtocol for MyFuture {
    // Now MyFuture has truly become a Future wrapper
    fn __next__(mut py_slf: PyRefMut<Self>) -> IterNextOutput<PyRefMut<Self>, u32> {
        // This is so we can borrow both slf.waker & slf.future mutably
        let slf = &mut *py_slf;
        let waker = slf.waker.as_mut().expect("future was not awaited");
        // Thanks to `futures` we don't need to implement the Waker pseudo-trait
        // manually, so this is all boiler-plate to pass the Waker to the Future.
        let waker_ref = waker_ref(&waker);
        let context = &mut Context::from_waker(&*waker_ref);
        match slf.future.as_mut().poll(context) {
            Poll::Pending => {
                // In case the future needs to be put on hold multiple times,
                // we need to set this back to true every time we're in a
                // Pending state. Asyncio always sets it to false after
                // reading it when we yield ourself.
                slf._asyncio_future_blocking = true;
                IterNextOutput::Yield(py_slf)
            }
            Poll::Ready(result) => IterNextOutput::Return(result),
        }
    }
}

#[pyclass]
#[derive(Clone)]
struct WakerClosure {
    aio_loop: PyObject,
    py_future: Py<MyFuture>,
}

// WakerClosure now implements `futures::task::ArcWake` so it can be passed to a
// Rust Future as a waker, like we did above.
impl ArcWake for WakerClosure {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        let closure = (**arc_self).clone();
        Python::with_gil(|py| {
            arc_self
                .aio_loop
                .call_method1(py, "call_soon_threadsafe", (closure,))
        })
        .expect("exception thrown by the event loop (probably closed)");
        // Again, as a reminder, we are only setting ourself up to be called by
        // the event loop from the main thread so this child thread is released
        // before the task is woken up. Remember how the TimerFuture still has
        // its shared state locked when calling `.wake()`? This is where it
        // would have become a problem if we tried to call the callbacks directly.
        // Of course, here, we could fix the issue in the TimerFuture's code, but
        // in the wild we won't have control over how the waker is called.
        // (That state locking issue isn't something I made up or introduced on
        // purpose, it's genuinely the code from the async book.)
    }
}

#[pymethods]
impl WakerClosure {
    // I didn't change anything here, but as an exercise, try and move the code
    // from this function to `wake_by_ref` above to skip the double callback,
    // and see what happens.
    #[call]
    pub fn __call__(slf: PyRef<Self>) -> PyResult<()> {
        let py = slf.py();
        let mut py_future = slf.py_future.try_borrow_mut(py)?;
        let callbacks = std::mem::take(&mut py_future.callbacks);
        for (callback, context) in callbacks {
            slf.aio_loop.call_method(
                py,
                "call_soon_threadsafe",
                (callback, &py_future),
                Some(vec![("context", context)].into_py_dict(py)),
            )?;
        }
        Ok(())
    }
}

#[pymethods]
impl MyFuture {
    fn get_loop(&self) -> Option<&PyObject> {
        self.aio_loop.as_ref()
    }

    fn add_done_callback(&mut self, callback: PyObject, context: Option<PyObject>) {
        self.callbacks.push((callback, context));
    }

    fn result(&self) -> Option<PyObject> {
        None
    }
}

// All that's left to do now is to wrap the TimerFuture in MyFuture, and everything
// works just as before.
// (You'll have to update the Python shim to use this function.)
#[pyfunction]
fn my_timer_future(result: u32) -> MyFuture {
    MyFuture::new(TimerFuture::new(result))
}

#[pymodule]
fn awaitable_rust(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_class::<MyFuture>()?;
    m.add_function(wrap_pyfunction!(my_timer_future, m)?)?;
    Ok(())
}
```

**And with that you have a working wrapper for asyncio-driven Rust futures.**

All that is left to do now is to generalise to any kind of future (by taking an `impl IntoPyObject` as `Output` for example), add a `cancel` method, potentially implement the `Waker` pseudo-trait manually for our `WakerClosure` to avoid wrapping it in an `Arc`, and generally polish some rough edges.

But that is left to libraries to implement, or to the reader as an exercise.
