---
title: Concurrent error handling
date: 2014-10-05
encoding: utf-8
layout: post
---

Unfortunately, many introductory programming materials across languages gloss
over the very real (and likely) possibility that a program will fail at some
point. Take this code, for example:

{% highlight go %}
func printContents(path string) {
    f, _ := os.Open(path)
    defer f.Close()

    s := bufio.NewScanner(f)
    for s.Scan() {
        fmt.Println(s.Text())
    }
}
{% endhighlight %}

This is a fairly ordinary code sample that one might find in a textbook or
useful blog post. The function opens a file, wraps a scanner around it, and
prints each line that the scanner detects. Overall the code is reasonably clear
and easy to comprehend, but misses a glaring problem: `os.Open` may return an
error instead of a file handle. Try to run this code with a bad path value and
it will panic almost immediately. Of course, we can fix that problem by just
passing the error up the chain:

{% highlight go %}
func printContents2(path string) error {
    if f, err := os.Open(path); err != nil {
      return err
    }

    defer f.Close()

    s := bufio.NewScanner(f)
    for s.Scan() {
        fmt.Println(s.Text())
    }

    return nil
}
{% endhighlight %}

Simple error handling like this is easy enough to implement, if not
particularly sophisticated. Now imagine using `printContents` in its own
goroutine:

{% highlight go %}
func someFunc() {
  // other stuff...

  path := "path-that-does-not-exist"

  go printContents2(path)

  // wait for this to finish...
}
{% endhighlight %}

In the first implementation of `printContents` we would panic if `os.Open`
returned an error. But here we have an altogether different - and potentially
more dangerous - problem. If `printContents2` returns an error, we won't know.
Outside of the goroutine there is no indication anything has gone wrong, likely
because there was no assumption in writing the code that anything could go
wrong. And this is where the issue of concurrent error handling begins.

Detecting errors in a serial programming environment is hard enough, but we
have the benefit of implicitly knowing that the order of program execution at
least approximates the order of actual machine instructions. In concurrent
environments there is no such luxury. When a new goroutine is created in Go,
all it shares with other goroutines is a common memory space. There is no
single stack, or notion of a caller to return data to. Instead the burden is on
the person writing code to communicate both what went right and what went wrong
during the life of a goroutine.

While error handling in general is complex and context-dependent, there are a
few useful patterns that can be used in situations where concurrent goroutines
can generate errors or fail outright.
