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
* In some high-level languages (Python !) we can use Coroutines to do that
  easily

## Synchronous I/O diagram

\begin{center}
 \includegraphics[witdh=12]{img/synchronous.png}
\end{center}

# Generators and coroutines

## Disclaimer

* Python >= 3.3

## Generators

\begin{minted}{python}
def make_generator():
    yield 0
    yield 1

g = make_generator()
\end{minted}

## Coroutines (yield)

## yield from (Python 3.3)
