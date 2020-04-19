---
layout: post
title: "Worker pools in Go using buffered channels and Goroutines"
date: 2020-04-19
author: Neil Alexander
index: true
---

[Goroutines](https://tour.golang.org/concurrency/1) in Go are effectively
lightweight threads. The Go runtime will schedule goroutines across a number of
real operating system threads (often based on the number of available CPU cores
on the system) so that they run concurrently on multi-core systems. It's common
in Go to communicate between goroutines using
[channels](https://tour.golang.org/concurrency/2), which are inherently
thread-safe.

In the case that you have lots of similar tasks to be performed but you want to
control the number of tasks that can be ran simultaneously, you can build a
worker pool. You may wish to do this to cap memory consumption, network traffic
or file descriptor use.

Start by defining a job, which contains all of the context required for a worker
to complete its work. In our simple example, this is simply `text` to print:

```
type job struct {
	text string
}
```

Create 30 sample job items which we will insert into our pending queue shortly:

```
jobs := []job{}
for i := 1; i <= 30; i ++ {
	jobs = append(jobs, job{
		text: fmt.Sprintf("Position %d in queue", i),
	})
}
```

Create the pending job queue, which is a [buffered
channel](https://tour.golang.org/concurrency/3). Using a buffered channel here
allows us to write into the channel without blocking. Important to note here is
that the buffered channel must be equal to or greater than `len(jobs)`,
otherwise any writes to the channel above the designated capacity will block.
Once we are done, we must `close()` the channel so that the workers will be able
to determine when there is no more work pending in the queue:

```
pending := make(chan job, len(jobs))
for _, j := range jobs {
	pending <- j
}
close(pending)
```

Define a maximum number of workers. If the number of pending jobs is less than
the maximum number of allowed workers then we should reduce the maximum number
of allowed workers to the number of pending jobs—this prevents us from starting
more workers than we need:

```
numWorkers := 5
if len(jobs) < numWorkers {
	numWorkers = len(jobs)
}
```

Create a wait group and add the number of workers to the wait group. This allows
us to block whilst waiting on the worker pool finishing all of the pending
tasks:

```
var wait sync.WaitGroup
wait.Add(numWorkers)
```

Define what the worker should do for each job. In our example, we will simply
print the `text` from within the `job` struct:

```
work := func(j job) {
	fmt.Println(j.text)
}
```

Then define the worker itself. The `range ch` here will continuously pull jobs
from the receive-only pending queue, as passed to the worker function, in a
thread-safe fashion. The `range ch` will automatically finish when there are no
more pending work objects in the buffered channel, reaching the channel closure
from above. Once the `range ch` loop ends, the `defer` statement will notify the
wait group that the worker is no longer running:

```
worker := func(ch <-chan job) {
	defer wait.Done()
	for j := range ch {
		work(j)
	}
}
```

Now that the worker pool has been prepared and there is work to be done waiting
in the pending queue, we can now start the required number of workers to
complete the work:

```
for i := 0; i < numWorkers; i++ {
	go worker(pending)
}
```

Finally, we will wait for the worker pool to finish. Each worker signals to the
wait group that they are no longer running when there is no more work to do. The
following statement will block until that happens:

```
wait.Wait()
```

In [this Go Playground sample](https://play.golang.org/p/BevRmtES7lV) based on
the above code, there is a larger number of queued objects and a `time.Sleep()`
in the `work` function. This helps to show that only five workers, as defined in
`numWorkers`, are running at a time, until there is no more work to be done.

You now have a worker pool!
