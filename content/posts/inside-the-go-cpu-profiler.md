---
title: "Inside the Go CPU profiler"
date: 2022-08-26T20:03:25+03:00
draft: true
---


# Introduction

It is safe to say that Go is one of its kind when it comes to profiling: Go runtime 
includes powerful and opinionated(!) profilers inside the runtime. In other languages like
Ruby, Python, or Node.js, the standard library contains profilers or at least a few APIs for writing 
profilers, but those are very limited in scope compared to what Go offers out of the box. In Go, the runtime is pretty generous regarding observability tools. If you want to
learn more about the types of profilers, and tracing tools that Go has, I highly recommend Felix Geisendörfer's
[The Busy Developer's Guide to Go Profiling, Tracing and Observability](https://github.com/DataDog/go-profiler-notes/blob/main/guide/README.md)

As a curious engineer, I love to dig deep on how things work at a low level and I have always wanted to teach 
myself how Go CPU profiler works under the hood. So I started teaching myself how it works and 
then it turned into sharing this journey in a blog post. There is no such time, I did not find myself learning new things
when I read Go runtime and this was no exception. 

# Basics

There are two types of profilers:
1. **tracing**: do measurements whenever an pre-defined event happens. e.g., function called, function exited...etc.
2. **sampling**: do measurements at regular intervals.

Go CPU profiler is a sampling profiler. There is also a [Go execution tracer](https://pkg.go.dev/runtime/trace) which is 
tracing profiler and traces certain events like acquiring a Lock, GC related events...etc. From a design point of view, sampling profilers usually consists of two 
basic parts:

1. **sampler**: triggers a callback at regular intervals and this callback collects profiling data(usually a stack trace).Different
profilers use different strategies to trigger the sampling interval.
2. **data collection**: this is where profiler collects its data: it might be memory consumption, call count. Basically any metric that can be associated with a stack trace.

# How other profilers work?

Linux `perf` uses `PMU`(Performance Monitor Unit) counters for sampling. In summary, you can instruct the PMU to generate an interrupt after some PMU event happens N times. An example might be to tick in every 1000 CPU clock cycles. There is a [detailed article](https://easyperf.net/blog/2018/06/01/PMU-counters-and-profiling-basics) written by Denis Bakhvalov that explains how tools like `perf`, `VTune` use PMU counters to make this happen. Once the data collection callback is triggered at regular intervals, all is left to collect stack traces and aggregate them properly. To be complete, Linux `perf` uses `perf_event_open(PERF_SAMPLE_STACK_USER,...)` to obtain stack trace information. The captured stack traces are written to userspace via mmap'd ring buffer...

[`pyspy`](https://github.com/benfred/py-spy) and [`rbspy`](https://github.com/rbspy/rbspy) are famous sampling profilers for Python and Ruby. They both ran as external processes and they periodically read the target application memory to capture the stack trace of running threads. In Linux, they use `process_vm_readv` and if I am not mistaken this API pauses the target application for few miliseconds during memory read. Then they chase pointers inside the memory they read to find the current running thread structure and stack trace information. As you might guess, this is a error prone and complex approach but works surprisingly well. IIRC, [`pyflame`](https://github.com/uber-archive/pyflame) uses similar approach too.

Recent profilers like [Parca](parca.dev)(there are few others) use [eBPF](https://ebpf.io/). eBPF is a recent technology that allows to run userland code in the Kernel VM. It is a brilliant technology that is used in lots of areas like security, networking and observability for sure. I highly suggest to read some information on eBPF, it is a massive topic that goes well beyond the scope of this blog post.

I think you get the idea: although there are various strategies to implement a sampling profiler under the hood, the underlying ideas/patterns stays same:
setup a periodic handler and read/aggregate stack trace data.

# How the profiler is triggered periodically?

Let's reiterate again: Go CPU profiler is a sampling profiler. In Linux, Go runtime uses `setitimer`/`timer_create/timer_settime` APIs to set up a `SIGPROF` signal handler. This handler is triggered at periodic intervals that is controlled by `runtime.SetCPUProfileRate` which is `100Mz(10ms)` by default. As a side note: surprisingly, there were some serious issues around the sampler of the Go CPU profiler until Go 1.18! You can see the gory details on the problems around it [here](https://www.datadoghq.com/blog/engineering/profiling-improvements-in-go-1-18/). IIUC, `setitimer` API was the recommended way of triggering time based signals per-thread in Linux but it was not working as advertised. Please feel free to correct me if this claim is wrong.

Let's see how you can enable the Go CPU profiler manually:

```go
f, err := os.Create("profile.pb.gz")
if err != nil {
    log.Fatal(...)
}
if err := pprof.StartCPUProfile(f); err != nil {
    ...
}
defer pprof.StopCPUProfile()
```

Once `pprof.StartCPUProfile` is called, Go runtime enables the `SIGPROF` signal to be generated at specific intervals. Kernel sends a `SIGPROF` signal to one of the **running** threads in the application. Since Go ises non-blocking I/O, goroutines that are waiting on I/O are not counted as **running** and these are not catched by the Go CPU Profiler. In fact, this was the basic reason of why [fgprof](https://github.com/felixge/fgprof) was implemented.

A picture is worth thousand words:

![SIGPROF signal in the Go runtime](/sigprof.png)

# How profiler collects data?

 Once, a random running goroutine receives a `SIGPROF` signal, it gets interrupted and signal handler runs. The stack trace of the interrupted goroutine is retrieved in the context of this signal handler and then saved into a [lock-free](https://preshing.com/20120612/an-introduction-to-lock-free-programming/) log structure along with the current [profiler label](https://rakyll.org/profiler-labels/)(Every captured stack trace can be associated with a custom label which you can later do filtering on). This special lock-free structure is named as `profBuf` and it is defined in [runtime/profbuf.go](https://github.com/golang/go/blob/master/src/runtime/profbuf.go) with a long and detailed explanation on how it works. To summarize it up: it is a **single-writer**, **single-reader** lock-free [ring-buffer](https://en.wikipedia.org/wiki/Circular_buffer) structure which highly resembles the one published [here](http://www.cse.cuhk.edu.hk/~pclee/www/pubs/ancs09poster.pdf). The writer is the profiler signal handler and the reader is a goroutine(`profileWriter`) that periodically reads this buffer and aggregates results to a final hashmap structure in a separate goroutine. This hashmap structure is named as `profMap` and defined in [runtime/pprof/map.go](https://github.com/golang/go/blob/master/src/runtime/pprof/map.go)

Here is a simple visualization on how this all fits together:

![SIGPROF handler](/sigprof_handler.png)

As can be seen, the final structure resembles the regular `pprof.Profile` object a lot: it is a a hashmap where the key is stack trace + label and the value is the number of times where this callstack is observed in the application.

When `pprof.StopCPUProfile()` is called, profiling stops and the `profileWriter` goroutine calls `build()` function which is implemented [here](https://github.com/golang/go/blob/aa4299735b78189eeac1e2c4edafb9d014cc62d7/src/runtime/pprof/proto.go#L348). This function is responsible for writing the `profMap` structure to the `io.Writer` object that is provided in the initial
`pprof.StartCPUProfile` call. Basically this is where the final [`pprof.Profile`](https://pkg.go.dev/runtime/pprof#Profile) object is generated. A pseudocode for `pprof.StopCPUProfile()` might be helpful here:

![pseudo_code](/pseudo_code.png)

Once I have a high-level understanding of this the overall design, the first question I asked to myself was following: 

*Why the Go runtime took all the trouble for implementing a unique lock-free structure just for holding temporary profiling data? Why not write everything to a hashmap periodically?*

I think the answer is in the design itself:

Look at the first thing that the `SIGPROF` handler does is to disable memory allocation. Addition to that, the profiler code path does not involve any locks and even the maximum depth of stack trace is hardcoded. As of `Go 1.19`, it is [64](https://github.com/golang/go/blob/54cf1b107d24e135990314b56b02264dba8620fc/src/runtime/cpuprof.go#L22). All these details leads to a more efficient and predictable profiler. Predictable is key here. It is important to have a constant overhead for a production-ready profiler. 

# Overhead

Based on the design, the profiler overhead should be deterministic. And this is *true*. Well, kind of... Let me explain. In a single profiler interrupt, following happens:

1. a random running goroutine context switch to run the `SIGPROF` handler code,
2. stack walk happens and saved to a lock-free ring buffer,
3. the goroutine is resumed.

While I could not find the reference, I remember all above happens in around ~10 nanoseconds. In theory, if you interrupt a Go application 100 times a second, the profiler overhead should be close to ~1000 nanosecs. Add the cycles spent for `ProfilerWriter` goroutine and you will have a predictable, constant number. But, that is not the case. If you search for the overhead of Go CPU profiler, you will see numbers ranging from %1-%5 and even in some rare cases %10. The reason behind this related with how CPUs work. Modern CPUs are complex beasts. They cache aggressively. A typical single CPU core has 3 layers of caches: L1, L2 and L3. When a specific CPU intensive code is running, these caches are highly utilized. This is especially true for some applications where a small, sequential(data that can fit in cache) amount of data is read heavily. A good example to this is matrix multiplication. During matrix multiplication, CPU heavily access individual cells which are sequential in memory. These kind of applications might produce *worst* behaviour for a sampling profiler. While it is tempting to do some benchmarks with `perf` to see CPU behavior for different type of applications, I think it is beyond the scope of this blog post.

In summary, there is a bit of uncertainty on the overhead of the profiler. But Go runtime has done a good job to keep it as predictable and as low as possible. If you don't believe me, which you should not, maybe below can convince you:

> At Google, we continuously profile Go production services and it is safe to do so.

Above is a quote from a [Google thread](https://groups.google.com/g/golang-nuts/c/e6lB8ENbIw8/m/azeTCGj7AgAJ).

And another one is from a [commit](https://github.com/DataDog/dd-trace-go/commit/54604a13335b9c5e4ac18a898e4d5971b6b6fc8c) from DataDog's continuos profiler implementation which makes the profiler to **always be enabled**:

> After testing this default on many high-volume internal workloads, we've
determined this default is safe for production. 

# Conclusion

One of the things I love about this design is that it proves how well you can optimize code depends on how well you understand the access patterns of your underlying data structures. In this case, a lock-free structure is used even although it is mostly a complete overkill. As mentioned in the beginning of this blog post, Go runtime is full of
clever optimizations like these and provides an excellent example of how and when to do optimizations.

Hope you enjoyed!

# References

xxx
