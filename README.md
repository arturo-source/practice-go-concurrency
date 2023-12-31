# Go concurrency

To start learning about go concurrency, I've chosen my trusted site <https://gobyexample.com/>. There you can see the simplest examples for the standard programming cases in Go. I recommend starting from **Goroutines** and reading all the examples until **Stateful Goroutines**.

Obviously you should also read the Go tour <https://go.dev/tour/concurrency/1>, which have some examples that are useful to start.

The last thing that you should take a quick look is this video <https://youtu.be/LvgVSSpwND8>. Jake Wright talks about WaitGroup, Channels, Deadlock and Channel closing, buffered Channels, Select statement, and the Worker pool pattern, this is a good introduction for Golang concurrency. **Old but gold**.

At this time, I recommend you to stop reading, and practice with some small problems (e.g. [Tree](01.%20Tree) and [Crawler](02.%20Crawler)).

After solving some small problems, you can continue watching the [Rob Pike conference](https://www.youtube.com/watch?v=f6kdp27TYZs) (or you can read [the slides](https://go.dev/talks/2012/concurrency.slide#1), but I recommend the video). Here he explains what is concurrency, some Go concurrency "patterns", and why Go concurrency was made as a part of the language. **Old but gold**.

If you want to get harder, read this post <https://go.dev/blog/pipelines> from the go official blog. This post takes a real example where you can apply concurrency, and breaks it down carefully, explaining why each decision is made. I have modified the problem to do a similar execire using Go's concurrency: [MD5 files](03.%20MD5%20files).

## To do

- I'd like to take a look at <https://go.dev/doc/articles/race_detector>, <https://github.com/golang/go/wiki/LearnConcurrency>. These are some important articles that I've found.
- What is the best way to control errors? In channels you only send one type of data.

## Frequently asked questions

Here I'll attach some questions I had during my learning process.

### Who should close the channel?

Always **the sender**. Trying to write a closed channel will panic and terminate the program. Trying to read a closed channel will return a nil value (it depends on the channel type: bool -> false, int -> 0, etc.). It's mentioned on [this page](https://go.dev/tour/concurrency/4) of the Go tour.

I also would like to add that if you have multiple senders, you should look for an alternative to close the channel. If one sender close the channel, and the other one tries to send more data, the program will panic.

### Should I close the channel?

**It's not necessary to close the channel**, the garbage collector will do (also mentioned in [the Go tour](https://go.dev/tour/concurrency/4)). Sometimes you will want to close the channel, for example when the reader uses a for-range, that will end the for loop. But the important thing is that **not closing the channel will not produce a memory leak**, you don't have to worry about that.

### How to know if a channel is closed

If you are a sender, you shoud not send data throught a closed channel, but there is not a way to know if channel is closed. If you are a receiver, it is really easy to check `v, isOpen := <-ch`, if the channel is closed `isOpen` will be false. But there is some syntactic sugar that will allow you to receive data until the channel is closed:

```go
func main() {
 c := make(chan int, 10)
 go fibonacci(c)
 for i := range c {
  fmt.Println(i)
 }
}
```

If you use the for-range golang style, this will end when the channel is closed. You will not have to worry about checking if channel is closed.

### What is a select?

**Select is a switch-like solution to wait for one of multiple channels**. Sometimes you will want to block the goroutine until one of multiple channels can run. This is the perfect case to use select. It's important to know that select accepts _reading_ case and _writing_ case.

```go
select {
case a <- 10: // a is writing the channel
  // Do something
case x := <- b: // b is reading the channel
  // Do something
}
```

But sometimes you will want to wait for multiple channels in an infinite for loop, to response all of them.

#### Be quiet with Select

If you run a `select` in a for loop, with a `default` case, the program may not act as you expected. This is an example **modified** from <https://go.dev/tour/concurrency/6>:

```go
package main

import (
 "fmt"
 "time"
)

func main() {
 tick := time.Tick(100 * time.Millisecond)
 boom := time.After(500 * time.Millisecond)
 for i:= 0; i < 4; i++ {
  select {
  case <-tick:
   fmt.Println("tick.")
  case <-boom:
   fmt.Println("BOOM!")
   return
  default:
   fmt.Println("    .")
  }
 }
}
```

This will cause:

```txt
    .
    .
    .
    .
```

But you will have a problem also with closed channels, that you may not expect:

```go
for {
  select {
  case msg1 := <-a:
    // Do something
  case msg2 := <-b:
    // Do something
  }
}
```

This seems a good way to block a goroutine until `a` or `b` receives data, infinitely. But if you know that `a` can be closed, you may think that the select will only choose `b` case, but **no**. `msg1` will be the `a` nil value (false for bool, 0 for int, etc.) and will be executed infinitely, and it will cause a 100% of use of your CPU. So you should think how to control this case.

### How to set timeouts?

This is clearly explained in <https://gobyexample.com/timeouts>. You can use `time` from the Go standard packages, which provide some useful functions like `After` or `Tick`. You can combine this functions with `select` to get what you want.

### Synchronize multiple goroutines

There are multiple ways to do this, but the more comprehensive one is [using WaitGroup](https://gobyexample.com/waitgroups). You could also [use a channel](https://gobyexample.com/channel-synchronization) but this come more difficult when you try to synchronize multiple goroutines.

### Important about Mutex and WaitGroup

Note that if you want to `wg.Done()` or `mu.Lock()` (for [WaitGroup](https://pkg.go.dev/sync#WaitGroup) and [Mutex](https://pkg.go.dev/sync#Mutex) respectively) you should always use the same, not a copy, using pointers or global variables. As said in the [sync package](https://pkg.go.dev/sync):

```txt
Values containing the types defined in this package should not be copied.
```

### Do goroutines and channels produce memory leaks?

Short answer: **channels no, goroutines yes**. As said before, channels are managed by the garbage collector, you should close channels if you want to communicate that you will not continue sending data. It's not the same for the goroutines, if you block a goroutine, don't stop an infinite for loop, or whatever that prevents the goroutine finishing, the garbage collector will not be able to close that goroutine. Let's see a simple example:

```go
package main

import (
 "errors"
 "fmt"
 "runtime"
 "time"
)

func NumGoroutines(when string) {
 fmt.Println(when, runtime.NumGoroutine())
}

func main() {
 defer NumGoroutines("After end:")
 NumGoroutines("Beginning:")
 errc := make(chan error) // channel len 0

 go func() {
  // Do some stuff that produces error
  err := errors.New("some error")
  errc <- err
 }() // If nobody reads errc (<-errc), goroutine keeps blocked and never ends...
 NumGoroutines("After starting a goroutine:")

 // The code continues...
 time.Sleep(time.Second)
}
```

With `runtime.NumGoroutine()` we can count active goroutines. With `defer` we execute that after the `main` end. Then, we start a goroutine that produces an error, we could expect that this error will be read, but if nobody reads errc, this goroutine will never end. That code produces the next output:

```txt
Beginning: 1
After starting a goroutine: 2
After end: 2
```

This problem would be solved using a buffer to errc `errc := make(chan error, 1)`. This don't block the goroutine, then it's finished, and you don't have to worry about `errc` because garbage collector will free `errc` memory when it detects it'll never be used.
