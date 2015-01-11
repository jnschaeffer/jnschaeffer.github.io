---
title: Concurrent error handling
date: 2015-01-10
encoding: utf-8
layout: post
---

(Note: This post builds somewhat on the Go pipelines pattern, found [here][1].)

Most resources regarding concurrency deal exclusively with code that can only
succeed. As almost anyone who has written code knows, however, this is rarely
the case. Code can fail for many reasons, from connection timeouts to invalid
values passed to mathematical functions. And in a concurrent environment where
computations may be executed independently with shallow stacks, tracking down
and handling errors can be a problem. Given this, when we have multiple steps
in our code that may fail, how can we make each step concurrent while cleanly
handling errors that may arise? There are a few approaches we can take to
accomplish this, each of which has its own benefits and drawbacks.

For this post, all code is available on [GitHub][2] in the `error-mux` package.

### Example: Weather lookup service

We will be using a simple example weather service designed to look up
temperatures for a given ZIP code. The weather service has three important
functions: `getName`, `getTemp`, and `getWeather`. As the names imply,
`getName` gets the name of the place designated by the ZIP code, while
`getTemp` gets the local temperature in degrees Celsius. `getName` and
`getTemp` are both I/O-bound tasks which can easily be run concurrently (a
sleep function here is used as a stand-in for a database query or web service
call). By making them run concurrently we can gain some performance
improvements, but the return types of the functions pose some problems:

{% highlight go %}
func getName(zipCode string) (name string, err error)

func getTemp(zipCode string) (tempC float64, err error)
{% endhighlight %}

Both `getName` and `getTemp` return error values along with their results. To
work with these effectively we will need to explore a few strategies for
handling errors while rewriting the code to run concurrently. We will start by
wrapping our utility function calls in anonymous function goroutines.

### 1. Wrap each step in an anonymous function

This is the simplest and most naive approach, but certainly not the most
robust, as we will soon see. The basic idea here is to take each step in our
function, wrap it in an anonymous function with a `sync.WaitGroup`, and then
check each result for errors. Thus, our function now looks like this:

{% highlight go %}
// weather-concurrent-1/main.go

func getWeather(zipCode string) (w *Weather, err error) {
	var (
		name             string
		tempC            float64
		nameErr, tempErr error
	)

	wg := sync.WaitGroup{}
	wg.Add(2)

	go func() {
		name, nameErr = getName(zipCode)
		wg.Done()
	}()

	go func() {
		tempC, tempErr = getTemp(zipCode)
		wg.Done()
	}()

	wg.Wait()

	switch {
	case nameErr != nil:
		err = nameErr
	case tempErr != nil:
		err = tempErr
	default:
		w = &Weather{Name: name, TempC: tempC}
	}

	return
}
{% endhighlight %}

You can see that `getName` and `getTemp` are each in anonymous functions now,
each of which assign their results to variables in the scope of `getWeather`.
We use the `sync.WaitGroup` to make sure everything is finished before we check
for errors, which takes place in the switch block at the end of the function.
This works, but it involves a lot of boilerplate code - writing `go func()`
gets tedious very quickly. Missing a `wg.Done()` statement will also cause an
immediate deadlock, stopping the program entirely. And as a final stylistic
point, the code itself violates a central tenet of concurrency in Go: to share
memory by communicating, not to communicate by sharing memory.

With these issues in mind, we can build a better solution that's more in line
with best practices in Go (and programming in general).

### 2. Return channels and check each one for errors

This time, the bulk of our refactoring will be in `getName` and `getTemp`, not
`getWeather`. Instead of wrapping the function calls in `getWeather` in
anonymous functions, we return separate channels for each function result and
any errors. `getName` and `getTemp` now look like so:

{% highlight go %}
// weather-concurrent-2/main.go

func getName(zipCode string) (<-chan string, <-chan error) {
	names := map[string]string{
		"19123": "Philadelphia, PA",
		"90210": "Beverly Hills, CA",
	}

	out := make(chan string, 1)
	errs := make(chan error, 1)

	go func() {
		time.Sleep(time.Second)
		if name, ok := names[zipCode]; ok {
			out <- name
		} else {
			errs <- fmt.Errorf("getName: %d not found", zipCode)
		}

		close(out)
		close(errs)
	}()

	return out, errs
}

func getTemp(zipCode string) (<-chan float64, <-chan error) {
	temps := map[string]float64{
		"19123": -5.0,
		"90210": 27.3,
	}

	out := make(chan float64, 1)
	errs := make(chan error, 1)

	go func() {
		time.Sleep(time.Second)
		if temp, ok := temps[zipCode]; ok {
			out <- temp
		} else {
			errs <- fmt.Errorf("getTemp: %d not found", zipCode)
		}

		close(out)
		close(errs)
	}()

	return out, errs
}
{% endhighlight %}

The anonymous function logic we were using before has been pushed into both
functions, each of which send the function results through one of two channels
rather than returning them directly. This, in turn, makes our logic for
`getWeather` much simpler, similar to the original implementation:

{% highlight go %}
// weather-concurrent-2/main.go

func getWeather(zipCode string) (w *Weather, err error) {

	nameOut, nameErr := getName(zipCode)
	tempOut, tempErr := getTemp(zipCode)

	var open bool

	if err, open = <-nameErr; open {
		return
	}
	if err, open = <-tempErr; open {
		return
	}

	w = &Weather{Name: <-nameOut, TempC: <-tempOut}

	return
}
{% endhighlight %}

In this version of `getWeather`, `open` is a boolean variable that will be
`true` if the channel being read is open.

This implementation is certainly better than the first approach, but the
repetition of `if err, open = ...; open` is less than ideal. As the number of
concurrent tasks grows, so will the number of `if` statements. Fortunately, we
can use the next (and final) approach discussed to fix this.

### 3. Multiplex error channels and check the result

Instead of overhauling any of our current functions, we'll be taking a bit of
inspiration from the pipeline pattern here to handle our errors. The pipeline
pattern introduces the `merge` function, which takes in a variable list of
channels and merges (multiplexes) them into a single channel. Instead of
merging channels of integers as shown in the example, however, we'll be merging
channels of errors:

{% highlight go %}
// weather-concurrent-3/main.go

func mergeErrors(chs ...<-chan error) <-chan error {
	out := make(chan error)
	wg := sync.WaitGroup{}

	for _, ch := range chs {
		wg.Add(1)
		go func(c <-chan error) {
			for e := range c {
				out <- e
			}
			wg.Done()
		}(ch)
	}

	go func() {
		wg.Wait()
		close(out)
	}()

	return out
}
{% endhighlight %}

The logic of `mergeErrors` is as follows. For each error channel a new
goroutine is created which will copy all values from that channel to the output
channel. When all input channels are closed, the output channel is also closed.
This means that instead of checking two (or many) channels, we simply need to
iterate over the merged channel and collect any errors encountered:

{% highlight go %}
// weather-concurrent-3/main.go

func getWeather(zipCode string) (w *Weather, err error) {

	nameOut, nameErr := getName(zipCode)
	tempOut, tempErr := getTemp(zipCode)

	errs := mergeErrors(nameErr, tempErr)

	for e := range errs {
		if err == nil {
			err = e
		}
	}

	if err != nil {
		return
	}

	w = &Weather{Name: <-nameOut, TempC: <-tempOut}

	return
}
{% endhighlight %}

You may notice we `range` over the entire channel instead of just pulling the
first value and returning. In Go, **any blocked goroutines will not be cleaned
up by the garbage collector**; if we return after getting only the first error,
we'll lose the reference to the multiplexed channel and cause a goroutine leak.
Do not do this.

While the presence of the loop over the error channel in our third method looks
a bit clunky, we can actually factor it out nicely and use a utility function
or method to handle the errors from our goroutines. For example, we may want to
merge all errors into one large error, or to check each error to see if it
requires stopping the entire function or can be ignored. Doing this also allows
us to make standardized, reusable logic for handling errors in concurrent
contexts.

Additionally, it is possible to remove the need to iterate over the entire
error channel using the early termination technique described in the pipeline
pattern post (e.g. a quit function to close the merged error channel). For the
sake of brevity this is left as an exercise for any interested reader.

### Moving forward

In this post, we have looked at the problem of error handling in concurrent
contexts and implemented logic to make concurrent error handling cleaner and
more flexible. Having good error handling logic in concurrent code is crucial,
as errors can appear at any time and may be many goroutines away (rather than
being many function calls deep). By creating what is effectively a pipeline of
errors, we can isolate the idea of handling many concurrent computations from
handling many concurrent failures. And by keeping error handling code clean and
reusable, we can in turn have safer and more robust code at little additional
cost.

[1]: http://blog.golang.org/pipelines "Pipelines"
[2]: https://github.com/jnschaeffer/blog-examples "GitHub"
