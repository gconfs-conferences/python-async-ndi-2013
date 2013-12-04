% Introduction to Asynchronous Programming in Python
% Antoine Pietri
% 2013-12-06

# Introduction

## Synchronous I/O

* In the vast majority of server applications, the limiting factor is I/O
* An I/O request is blocking by default
* All the CPU time is wasted waiting for the request to complete

## Synchronous I/O diagram

\begin{center}
 \includegraphics[witdh=12]{img/synchronous.png}
\end{center}

## Note on Synchronous I/O

* There are some ways of making synchronous I/O non blocking
* fcntl() with the O\_NONBLOCK flag allows non-blocking read/write requests
* Synchronous non/blocking I/O is a perfectly valid way to make efficient I/O
* But that's not the subject of this presentation anyway

## Asynchronous I/O

* We need a way to keep the CPU busy
* We can't afford to waste all the CPU time waiting for some I/O requests to
  complete
* We could use asynchronous blocking I/O (select(2), poll(2), epoll(7),
  kqueue(7)), but that's tedious
* In some high-level languages (Python!) we can use Coroutines to do that
  easily

## Synchronous I/O diagram

\begin{center}
 \includegraphics[witdh=12]{img/synchronous.png}
\end{center}

# Generators and coroutines

## Disclaimer

* Python >= 3.3
* PEP 380
* yield from
* Returning from a function

## Generators

\begin{minted}{python}
    def make_generator():
        yield 0
        yield 1

    g = make_generator()

    print(g)
    # <generator object make_generator at 0x7fe98ec66690>

    print(next(g))
    # 0
    print(next(g))
    # 1
    print(next(g))
    # Exception: StopIteration
\end{minted}

## Generators

\begin{minted}{python}
    def make_generator():
        print('Start')
        yield 0
        print('Next')
        yield 1
        print('End')

    g = make_generator()

    print(next(g))
    # Start
    # 0
    print(next(g))
    # Next
    # 1
    print(next(g))
    # End
    # Exception: StopIteration
\end{minted}

## Generators

Easy way to have "laziness" in Python:

\begin{minted}{python}
    def get_pages():
        for i in range(10):
            yield requests.get('http://example.com/page/{}'.format(i)).text

    def main():
        for p in get_pages():
            print(p)
\end{minted}

## Coroutines

* A coroutine is a routine that can be suspended and resumed
* Cohabits with the routines, with a different context stack
* Very handful for Asynchronous I/O
* Added in Python in PEP 342 ("Coroutines via enhanced generators")

## Coroutines

\begin{center}
 \includegraphics[witdh=12]{img/coroutine\_diagram.png}
\end{center}

## Coroutines

Let's take our previous example:

\begin{minted}{python}
    def main():
        for p in get_pages():
            print(p)
\end{minted}

## Coroutines

It can be rewritten like this:

\begin{minted}{python}
    def main():
        generator = get_pages()
        while True:
            try:
                print(next(generator))
            except StopIteration:
                break
\end{minted}

* We can see the explicit call to next(), which *resumes* the coroutine.
* But maybe we could send something, when resuming this coroutine?

## Coroutines

This is equivalent to:

\begin{minted}{python}

    def main():
        generator = get_pages()
        while True:
            try:
                print(generator.send(None))
            except StopIteration:
                break

\end{minted}

* A generator is a coroutine
* A call to next() is a call to .send() with the None value.

## Coroutines

* We can communicate with our coroutine by sending values!
* Every generator can be resumed with their method send(value)

## Coroutines

\begin{minted}{python}
    def make_coroutine():
        val = yield 1
        print('coroutine got {}'.format(val))
        val = yield 2
        print('coroutine got {}'.format(val))

    def main():
        coroutine = make_coroutine()
        i = coroutine.send(None) # starts coroutine
        while True:
            try:
                print('main() retrieved {}'.format(i))
                print(coroutine.send(i * 2))
            except StopIteration:
                break
\end{minted}

## yield from

\begin{minted}{python}
    yield from coroutine
\end{minted}

* Added in Python 3.3 (PEP 380)
* Acts as a "pipe" between coroutines

## yield from

\begin{minted}{python}
    def coroutine1():
        a = 0
        while True:
            a = yield a
            a += 10

    def coroutine2():
        yield 42
        yield 1337
        yield from coroutine1()

    def main():
        a = 3
        c = coroutine2()
        while True:
            try:
                a = c.send(a)
                print(a)
            except StopIteration:
                break
    # 42, 1337, 3, 13, 23...
\end{minted}

# Coroutines for Async I/O

## Explanation

* So, how coroutines and Async I/O are actually related?
* The other common way of making Asynchronous I/O is using callback, but that's
  pretty verbose
* In each case, you need to run an event-loop somewhere.

## Callbacks

\begin{minted}{python}
    content = b''

    def read_async(fd):
        read(fd, callback=on_read_received)
        print('Not blocking!')

    def on_read_received(fd, data):
        print('Received data on fd {}'.format(fd))
        content += data

\end{minted}

* The framework provides base functions (read/write/...) that uses internally
  non-blocking synchronous I/O in their eventloop (kqueue, epoll, fallback on
  select)
* This is extremely verbose, due to the fact we're constantly changing routines
* At least we achieved our goal! We can launch "parallels" reads, instead of
  waiting at each blocking read.

## Callbacks

\begin{minted}{python}
    read_async(10) # (takes a fd)
    read_async(1)
    read_async(3)
    print('You see, not blocking !')

    # You see, not blocking !
    # Received data on fd 1
    # Received data on fd 10
    # Received data on fd 3
\end{minted}

## Futures

* What if we used **yield points** instead of callbacks?
* Instead of taking a callback parameter, functions of the framework returns
**futures**.
* When you yield a future, you "give back the hand" to your event loop
* The event loop resumes your coroutine when the result is ready

## Futures

\begin{minted}{python}
    content = b''
    def read_async(fd):
        future = read(fd, callback=on_read_received)
        print('Not blocking!')

        data = yield future # "Yield point", interrupt/resume
        content += data
\end{minted}

## Futures

* The I/O is no longer blocking, your coroutine is just "resumed" when the data
  is ready
* It means you can do effective I/O, without wasting your CPU time waiting for
  blocking I/O

## Futures

In the case of a web server:

\begin{minted}{python}
    handle_GET_request(sloooow_client)
    handle_GET_request(fast_client)
    handle_GET_request(not_very_fast_client)
    handle_GET_request(fast_client)

    # fast_client request handled!
    # fast_client request handled!
    # not_very_fast_client request handled!
    # sloooow_client request handled!
\end{minted}

# Frameworks

## Gevent

* Certainly the most known/used event library in Python 2
* No official support for Python 3, although some ports exist
* Weird implementation (x86 CPython specific bytecode)
* Monkey-patching: not nicely behaving with the standard library
* Everything is async by default: no explicit yield-points

## Twisted

* Good alternative, using "Deferred" for futures
* Similar mecanism
* A bit bloated
* Very hard to understand at first

## Tornado

* Web-oriented framework
* Very efficient
* Long-polling, websockets, HTTP streaming...
* Relies heavily on callbacks by design
* Light coroutine "syntastic sugar" have been around for a while

## Toro

* A library that provides synchronisation primitives for Tornado coroutines
* A lot of gevent ideas ported to Tornado
* Toro.Event(), Toro.Queue(), ...

## Tulip (asyncio)

* PEP 3156: Asynchronous IO Support Rebooted
* Goal: standardize the event-loop in Python to allow interoperability between
  frameworks
* Relies a lot on **yield from** to communicate with the event-loop.

\begin{minted}{python}
    @tulip.coroutine
    def download(url):
        response = yield from http.request('GET', url)
        data = yield from response.read()
\end{minted}
