---
title: Decoding external data with encoding/json
date: 2014-10-19
encoding: utf-8
layout: post
---

One particularly useful tool in the standard Go library is the
[encoding/json](http://golang.org/pkg/encoding/json) package. `encoding/json`
makes use of reflection and sensibly implemented interfaces to convert data
to and from JSON formats, requiring in most cases little more than an import
and a call to `json.Unmarshal`.

In general, the `encoding/json` package is designed for cases where the
structure of a type more or less matches the incoming JSON data. Unfortunately,
this isn't always the case. Many external APIs provide nonstandard or poorly
designed results for a given request, making conversion from JSON difficult.
Consider the following type declaration for a temperature reading:

{% highlight go %}

// Temperature represents a single temperature reading in Celsius at a given
// time.
type Temperature struct {
    DegC     float64
    // ReadTime comes from the time package: http://golang.org/pkg/time#Time
    ReadTime time.Time
}

{% endhighlight %}

This is a reasonable implementation for a single reading. But imagine our data
source only gives us JSON readings of the following form:

{% highlight json %}
{
  "reading": {
    "temp_c": 25.6,
    "unixtime": 1413767259
  }
}
{% endhighlight %}

What seemed like a good implementation for `Temperature` is now completely
incompatible with the data we're given. There are lots of approaches we could
take to handle this - wrapper types for `Temperature`, wrapper functions around
`json.Unmarshal`, or even our own parser - but what we really want to do is
contain the unmarshaling logic such that an end user can call `json.Unmarshal`
for a `Temperature` or anything that contains a `Temperature` and not need to
do any extra work beyond the standard `encoding/json` functions. We'll explore
a few ways to handle each individual problem with the JSON we're given and put
it all together into a cohesive example at the end.

## Unmarshaling with struct tags
There are two obvious problems with the JSON we have now: the actual data we
want is inside a field called `reading`, and the field names in the objects
don't match our struct's field names. We can start by taking the naive approach
and defining a struct that matches the data we receive:

{% highlight go %}

// Temperature represents a single temperature reading in Celsius at a given
// time.
type Temperature struct {
    temp_c   float64
    unixtime int64
}

// TemperatureWrapper is a wrapper around a Temperature.
type TemperatureWrapper struct {
    reading Temperature
}

{% endhighlight %}

In addition to being unpleasant to look at and poor practice - we don't want
the structure of our data to be reliant on the structure of an external API -
this simply won't work. `encoding/json` will only unmarshal data into exported
fields. Try running the following code with the above types added:

{% highlight go %}

package main

import (
    "encoding/json"
    "fmt"
    "log"
)

// Temperature and TemperatureWrapper definitions here

func main() {
    b := []byte(`{"reading": {"temp_c": 25.6, "unixtime": 1413767259}}`)

    var tw TemperatureWrapper
    if err := json.Unmarshal(b, &tw); err != nil {
        log.Fatal(err)
    }

    fmt.Println(tw)
}

{% endhighlight %}

The output will be `{% raw %}{{0 0}}{% endraw %}`; a clear indication that no
unmarshaling ever took place.

Fixing this is easy, though. As explained in the documentation for
[`json.Marshal`](http://golang.org/pkg/encoding/json/#Marshal), we can use
struct tags to map JSON fields to Go struct fields:

{% highlight go %}

// Temperature represents a single temperature reading in Celsius at a given
// time.
type Temperature struct {
    DegC     float64 `json:"temp_c"`
    ReadTime int64   `json:"unixtime"`
}

// TemperatureWrapper is a wrapper around a Temperature.
type TemperatureWrapper struct {
    Reading Temperature `json:"reading"`
}

{% endhighlight %}

Redefine the types above as follows and our program will print
`{% raw %}{{25.6 1413767259}}{% endraw %}`, meaning our JSON was unmarshaled
successfully.

This solution has one critical flaw, though: when we unmarshal the JSON,we get
a `TemperatureWrapper` instead of a `Temperature`. Imagine we receive a data
series like so:

{% highlight json %}

[
  {"reading": {"temp_c": 25.6, "unixtime": 1413767259}},
  {"reading": {"temp_c": 25.7, "unixtime": 1413767260}},
  {"reading": {"temp_c": 25.5, "unixtime": 1413767261}},
  {"reading": {"temp_c": 25.8, "unixtime": 1413767262}}
]

{% endhighlight %}

We won't be able to unmarshal this into a `[]Temperature` like we might want.
Instead, we will need to unmarshal into a `[]TemperatureWrapper` and use some
extra logic to access the underlying `Temperature` objects and put them into a
slice. Fortunately, there is an alternative. Using type embedding and type
aliases, we can unmarshal the entire object defined by `TemperatureWrapper`
into a `Temperature` without ever leaving the bounds of `encoding/json`.

## Unwrapping inner JSON objects

For cases where the default behavior of `json.Unmarshal` does not work for us,
`encoding/json` provides an `Unmarshaler` interface type. If a given type
implements `Unmarshaler`, that type's `UnmarshalJSON` method will override the
default logic when unmarshaling into a field or variable of that type. We can
implement this interface on `Temperature` to unmarshal our provided JSON more
cleanly:

{% highlight go %}

// Temperature represents a single temperature reading in Celsius at a given
// time.
type Temperature struct {
    DegC     float64 `json:"temp_c"`
    ReadTime int64   `json:"unixtime"`
}

func (t *Temperature) UnmarshalJSON(b []byte) (err error) {
    var tw temperatureWrapper
    if err = json.Unmarshal(b, &tw); err != nil {
        return
    }

    *t = Temperature(tw.Reading)

    return
}

// temperatureJSON is an alias to Temperature that does not implement
// Unmarshaler.
type temperatureJSON Temperature

// temperatureWrapper is a wrapper around a Temperature.
type temperatureWrapper struct {
    Reading temperatureJSON `json:"reading"`
}

{% endhighlight %}

There are a few important things to note here. The first is that our
`temperatureWrapper` type is now unexported. It is good practice in Go (and any
other language with similar functionality) to not export more data than is
necessary. 

The next important point is that `temperatureWrapper.Reading` is no
longer a `Temperature`, but rather a `temperatureJSON`. This is because we have
defined `json.Unmarshaler` on our `Temperature`: if we were to leave the field
as a `Temperature`, we would hit infinite recursion during `json.Unmarshal`
until the program crashed as `Temperature.UnmarshalJSON` tried to call itself.
Aliasing `Temperature` as `temperatureJSON` avoids this problem because
`temperatureJSON` has no unmarshaling logic defined.

Finally, you may notice that `UnmarshalJSON` is defined not on a `Temperature`,
but a `*Temperature`. This is because we are not initializing a new
`Temperature` object, but rather passing a pointer to one and populating it
based on the binary JSON we are given.

As it stands, our new implementation is far more robust than the old one. To
further encapsulate our unmarshaling logic, we can also define our temporary
types inside `UnmarshalJSON`:

{% highlight go %}

func (t *Temperature) UnmarshalJSON(b []byte) (err error) {

    type temperatureJSON Temperature

    type temperatureWrapper struct {
        Reading temperatureJSON `json:"reading"`
    }

    var tw temperatureWrapper
    if err = json.Unmarshal(b, &tw); err != nil {
        return
    }

    *t = Temperature(tw.Reading)

    return
}

{% endhighlight %}

We have now completely contained the logic to unmarshal the inner temperature
object inside `UnmarshalJSON`, which is a great advantage in itself. However,
one problem remains. Our original definition of `Temperature` defined
`ReadTime` as a `time.Time`, but we only have a Unix timestamp in the form of
an `int64`. How can we handle this?

## Unmarshaling into embedded types

In most settings, one might convert a Unix timestamp to a time value by using a
polymorphic constructor or a getter method that converts the timestamp to a
better data structure. While we could easily define a `Temperature.ReadTime()`
method that would accomplish the latter, it would be better if we could simply
convert the Unix timestamp to a Go `time.Time` during unmarshaling. We can
accomplish this in a naive fashion by expanding `temperatureJSON` to be an
entirely new struct:

{% highlight go %}

// Temperature represents a single temperature reading in Celsius at a given
// time.
type Temperature struct {
    DegC     float64
    ReadTime time.Time
}

func (t *Temperature) UnmarshalJSON(b []byte) (err error) {

    type temperatureJSON struct {
        DegC     float64 `json:"temp_c"`
        ReadTime int64   `json:"unixtime"`
    }

    type temperatureWrapper struct {
        Reading temperatureJSON `json:"reading"`
    }

    var tw temperatureWrapper
    if err = json.Unmarshal(b, &tw); err != nil {
        return
    }

    *t = Temperature{}
    t.DegC = tw.Reading.DegC
    t.ReadTime = time.Unix(tw.Reading.ReadTime, 0)

    return
}

{% endhighlight %}

This works, but it has a bad code smell to it. We now have the `DegC` field of
`Temperature` repeated in `temperatureJSON`, and will need to do the same for
any other fields that may get added (or which we may want to add) to the
temperature JSON later. If possible, we would like to avoid this kind of
repetition in our code. And thanks to Go's type embedding mechanism, we can.

In place of traditional inheritance by subclassing, Go provides a mechanism
called inheritance by composition, also known as
[embedding](http://golang.org/doc/effective_go.html#embedding). For our JSON
unmarshaling problem, we can refactor our `temperatureJSON` type into a
`temperatureUnixJSON` that embeds a `temperatureJSON` and overrides only the
parsing of our read time:

{% highlight go %}

// Temperature represents a single temperature reading in Celsius at a given
// time.
type Temperature struct {
    DegC     float64   `json:"temp_c"`
    ReadTime time.Time
}

func (t *Temperature) UnmarshalJSON(b []byte) (err error) {

    type temperatureJSON Temperature

    type temperatureUnixJSON struct {
        // Note the embedding here.
        temperatureJSON
        ReadTime int64  `json:"unixtime"`
    }

    type temperatureWrapper struct {
        Reading temperatureUnixJSON `json:"reading"`
    }

    var tw temperatureWrapper
    if err = json.Unmarshal(b, &tw); err != nil {
        return
    }

    *t = Temperature(tw.Reading.temperatureJSON)
    t.ReadTime = time.Unix(tw.Reading.ReadTime, 0)

    return
}

{% endhighlight %}

During unmarshaling, our Unix timestamp will be unmarshaled into the
`temperatureUnixJSON.ReadTime` field. Our actual temperature, however, will be
unmarshaled into the embedded `temperatureJSON` field, bypassing the wrapper
objects entirely. By embedding the `temperatureJSON` (which is just an alias
for `Temperature`) in the `temperatureUnixJSON` type, we ensure that only the
fields defined in `temperatureUnixJSON` are unmarshaled into that type. The
rest will be unmarshaled into `temperatureJSON`, meaning if we were to add a
field like `StationID` or `DegF` in the future we could do so without changing 
the implementation of `Temperature.UnmarshalJSON`.

## Putting it all together

In this post, we have defined a simple struct representing a temperature
reading in degrees Celsius and further defined custom unmarshaling logic to
pull this reading from a messy JSON object using interfaces, type synonyms, and
type embedding. Situations like this are exceedingly common when dealing with
data from external sources. Often we don't have control over the structure of
the data we receive, and the data we get is likely to conflict somehow with the
structure we would like it to have.

When these problems are left unchecked, we can easily get lost in a mess of
wrapper types, custom getter methods, and code rot that extends far beyond any
initial unmarshaling logic. But by leveraging some of the features of Go's
type system and the `encoding/json` package, we can ensure that any problems we
encounter in dealing with serialized JSON data are largely self-contained.
