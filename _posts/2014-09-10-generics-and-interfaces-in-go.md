---
title: Generics and interfaces in Go
date: 2014-09-10
encoding: utf-8
layout: post
---

Go doesn't have generics, and won't for the foreseeable future. For lots of
people that's a pretty big disadvantage of the language; on the surface,
generics enable a programmer to adhere better to DRY (Don't Repeat Yourself)
principles and think about what they're writing in more general terms. But the
language has some powerful tools under its belt, and by leveraging them in the
right ways we can achieve everything generics can for solving problems, many
in more elegant and composable ways than object-oriented interfaces can
provide.

In object-oriented programming, a function operating on generics will typically
either operate on a stand-in for some single type (e.g. Java's `Iterator<E>`)
or a type that implements some interface (e.g. `SomeClass<E extends IFace>`).
While the intent in the first case is obvious, the intent in the second case is
more important here - we want any objects of type `E` to also have constraints
on what it is they actually do. In effect, we try to define the problem at two
ends. On one end, we have the abstract definition of `SomeClass`. On the other,
we have its more concrete implementation in `E` as defined by `IFace`. Java's
`Comparable<T>` is a good example of this approach: Objects may be of any type,
but any object of a class that implements `Comparable<T>` can be ordered and
sorted simply by having that implementation.

In Go we don't have this, of course. There are interfaces, and so we could
implement a `Comparable` interface of our own, but without knowing if any
two things we're comparing are of the same concrete type we're left with code
that might not actually work. Take the following example:

{% highlight go %}

package main

import "fmt"

type Comparable interface{
    CompareTo(c Comparable) bool
}

type MyInt int

func (i MyInt) CompareTo(c Comparable) bool {
    return i < c.(MyInt)
}

type MyString string

func (s MyString) CompareTo(c Comparable) bool {
    return s < c.(MyString)
}

func main() {
    var i MyInt = 10
    var s MyString = "hello"

    // what happens here? 
    fmt.Println(i.CompareTo(s))
}
{% endhighlight %}

It might be a simple program, but it illustrates the above point well. In the
last line, we're comparing a `MyInt` to a `MyString`. The code compiles, but
when we run it the runtime panics:

{% highlight text %}
    interface conversion: main.Comparable is main.MyString, not main.MyInt
{% endhighlight %}

Not good. So where does that leave us? The answer, it turns out, is that it
leaves us in a pretty good position.

One of the side effects of generics is that they promote code splitting into
*abstract* and *concrete* domains. In the abstract domain, we have high-level
types like collections of objects (arguably the kind of types where generics
are used most heavily). In the concrete domain, we have the implementation
logic that the abstract domain relies on, such as the ordering of instances of
a given type. What we get as a result is something like a pile of boxes with
very specifically shaped holes and a separate pile of very specifically shaped
pegs.

When we only deal with the very abstract and very concrete, it may seem like
there's nothing in the middle at all. But there is, and that space between is
where Go's data model comes in. Consider the problem of sorting a list of
objects. In a generic-powered, object-oriented world, we might have something
like a `Collections` class which sorts on a `List<T extends Comparable>` object
([approximately](http://docs.oracle.com/javase/7/docs/api/java/util/Collections.html#sort\(java.util.List\))
the way Java handles it). To put it in simpler terms, we're sorting on a list
of objects with some defined ordering. Now in contrast to this we have Go's
[`sort.Interface`](http://godoc.org/sort#Interface) type:

{% highlight go %}

// A type, typically a collection, that satisfies sort.Interface can be sorted
// by the routines in this package. The methods require that the elements of
// the collection be enumerated by an integer index.

type Interface interface {
    // Len is the number of elements in the collection.
    Len() int
    // Less reports whether the element with
    // index i should sort before the element with index j.
    Less(i, j int) bool
    // Swap swaps the elements with indexes i and j.
    Swap(i, j int)
}

{% endhighlight %}

The most striking feature of `sort.Interface` - besides the unfortunate
decision to name an interface type `Interface` - is that there isn't any
mention of what is actually being sorted. In fact, `sort.Interface` doesn't
require that any collection of objects exists at all. All it requires is that
there exists a mapping from an integer an ordering, and an ability to redefine
that ordering. What the things being ordered are has no impact on the actual
sorting process, and so they aren't included in the interface definition. 

The biggest problem with this implementation is that in practice, developers
using `sort.Interface` will find themselves rewriting `Len` and `Swap` for
every slice they use (note that implementations already exist for `[]int` and
`[]string`):

{% highlight go %}
type MyIntSlice []int

func (s MyIntSlice) Len() int {
    return len(s)
}

func (s MyIntSlice) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}

type MyStringSlice []string

func (s MyStringSlice) Len() int {
    return len(s)
}

func (s MyStringSlice) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}

{% endhighlight %}

This is a frustrating violation of keeping code DRY, even if it is only a
handful of lines of code. But we get a lot in exchange for a little redundancy.

Using `sort.Interface` as an example, we can see that it's easy, and in fact
necessary, to build an interface with a problem domain in mind. This stands in
contrast to the typical notion of an interface as providing a common set of
behaviors for some set of types. Instead, we provide the portions of the
problem logic that can be parameterized; logic that sits between the idea of
sorting and the idea of the thing to be sorted, or between the abstract and
concrete.

Because what we abstract away is the problem logic rather than object
behaviors, we can also implement the problem logic outside of the types we use
to solve the problem. There's no reason we need to bind the notion of ordering
and swapping to a slice of objects or the objects themselves. We can just as
easily create a type synonym that implements the logic, or make a wrapper type:

{% highlight go %}

package main

import (
    "fmt"
    "sort"
)

type MyIntSlice []int

func (s MyIntSlice) Len() int {
    return len(s)
}

func (s MyIntSlice) Less(i, j int) bool {
    return s[i] < s[j]
}

func (s MyIntSlice) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}

type MyIntSorter struct {
    Ints []int
}

// implementation left as an exercise for any interested reader.

func main() {
    list := []int{2, 9, 3, 7, 1}
    sort.Sort(MyIntSlice(list))
    fmt.Println(list)
}

{% endhighlight %}

Taking the approach of framing types in terms of the problems they solve also
enables us to readily take advantage of another feature of Go: interface and
struct composition. In Go, interfaces can be defined in terms of other
interfaces, like so:

{% highlight go %}

package main

import (
    "sort"
)

type Printer interface {
    PrintAt(i int)
}

type OrderedPrinter interface {
    sort.Interface
    Printer
}

// try coming up with an implementation of OrderedPrinter for a []int.

func printInOrder(p OrderedPrinter) {
    sort.Sort(p)

    for i := 0; i < p.Len(); i++ {
        p.PrintAt(i)
    }
}

func main() {

}

{% endhighlight %}

The intent of `OrderedPrinter` is clear and concise. Using only two lines
inside the interface body, we have said that we need all of the functionality
of printing and all of the functionality of sorting in the same type. Better
still, printing and sorting can come from different types that we can roll
into one:

{% highlight go %}

type MyIntPrinter struct {
    Vals []int
}

func (p MyIntPrinter) PrintAt(i int) {
    fmt.Println(p.Vals[i])
}

type MyIntSorter struct {
    Vals []int
}

// ...

type MyIntPrinterSorter struct {
    MyIntPrinter
    MyIntSorter
}

func NewIntPrinterSorter(vals []int) MyIntPrinterSorter {
    return MyIntPrinterSorter{MyIntPrinter{vals}, MyIntSorter{vals}}
}

{% endhighlight %}

If you write an implementation for `MyIntSorter` and try to pass a
`MyIntPrinterSorter` into the previously described `printInOrder` function,
you'll see that the result is just the same as if you had one concrete type
implementing both printing and sorting logic. And in a sense that's what we
have here as well - the trick is just that the implementing concrete type is
composed of other concrete types.

That line of thinking reflects one of the powerful principles underlying the
language. In Go, interfaces and composition let us work with concrete
implementations of low- and high-level code, using interfaces to communicate
between them only what needs to be communicated. The sort interface gives away
almost nothing about what it contains, yet gains enormously from the little it
provides. Similarly, our `MyIntPrinter` and `MyIntSorter` types define the
logic necessary to solve a problem and nothing more, but when composed together
are able to be used in more situations than either type could before.

Thinking in this way - about what something needs to *do* rather than what it
needs to *be* - helps keep code from migrating to the endlessly abstract or the
overly concrete with no room between. Instead, like channels between concurrent
routines, we can use purpose-driven interfaces to provide the minimum
information needed to solve a problem at a scale appropriate for the problem
domain. Doing so gives us code that is cleaner, more composable, and an overall
better reflection of what it is our code is actually meant to do.
