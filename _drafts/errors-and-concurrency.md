---
title: Generics and interfaces in Go
date: 2014-09-23
encoding: utf-8
layout: post
---

Detecting errors in a serial programming environment is hard enough, but we
have the benefit of implicitly knowing that the order of program execution at
least approximates the order of actual machine instructions. In concurrent
environments there is no such luxury.

The greatest benefit of concurrent programming is that we can compose many
units of computation so that they can function independently, but this also
leads to its greatest drawback: in a concurrent program we don't always know
when things fail. And if we don't build in proper communication between
concurrent processes, we may not know a failure has happened at all. Take this
code, for example:

{% highlight go %}
func printContents(path string) {
    if f, err := os.Open(path); err != nil {
        s := bufio.NewScanner(f)
        for s.Scan() {
            fmt.Println(s.Text())
        }
    }
}
{% endhighlight %}
