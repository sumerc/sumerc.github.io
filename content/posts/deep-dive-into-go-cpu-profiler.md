---
title: "Deep dive into the Go CPU profiler"
date: 2022-08-26T20:03:25+03:00
draft: true
---

# intro

It is safe to say that Go is one of its kind when it comes to profiling: Go runtime 
includes powerful and opiniated(!) profilers inside the runtime. In other languages like
Ruby, Python or Node.js there are some basic profilers and few APIs for writing 
profilers. In Go, however, runtime is pretty generous when it comes to observability tools. If you want to
learn more about the types of profilers, tracing tools that Go has, I highly recommend Felix Geisendörfer's
[The Busy Developer's Guide to Go Profiling, Tracing and Observability](https://github.com/DataDog/go-profiler-notes/blob/main/guide/README.md)

As a curious engineer, I love to dig deep on how things work at a low level and I have always wanted to teach 
myself how Go CPU profiler works under the hood. So I started teaching myself how it works and 
then it turned into sharing this journey in a blog post. There is no such time, I did not find myself learning new things
when I read Go runtime and this was no exception. 

# basics

There are two types of profilers:
 1. tracing : do measurements whenever an pre-defined event happens. e.g., function called, function exited...etc.
 2. sampling : do measurements at regular intervals.

 Go CPU profiler is a sampling profiler. There is also a [Go execution tracer](https://pkg.go.dev/runtime/trace) which is 
 tracing profiler and traces certain events like acquiring a Lock, GC related events...etc

Every sampling profiler consists two basic parts:
1. sampler
2. data collection

Sampler triggers a callback at regular intervals and this callback collects profiling data(usually a stack trace).Different
profilers use different strategies to trigger the sampling interval.

# few examples

Linux `perf` uses `PMU` counters to trigger the data collection. There is an [awesome article](https://easyperf.net/blog/2018/06/01/PMU-counters-and-profiling-basics) written by Denis Bakhvalov that explains how tools like `perf`, `VTune` use PMU counters to make this happen. 
Once the data collection callback is triggered at regular intervals, all is left to collect stack traces and aggregate them properly. To be complete, Linux `perf` uses `perf_event_open(PERF_SAMPLE_STACK_USER,...)` to obtain stack trace information. The captured stack traces are written to userspace via mmap'd ring buffer...

[`pyspy`](https://github.com/benfred/py-spy) and [`rbspy`](https://github.com/rbspy/rbspy) are famous sampling profilers for Python and Ruby. They both ran as external processes and they periodically read the target application memory to capture the stack trace of running threads. In Linux, they use `process_vm_readv` and if I am not mistaken this API pauses the target application for few miliseconds during memory read. Then they chase pointers inside the memory they read to find the current running thread structure and stack trace information. As you might guess, this is a error prone and complex approach but works surprisingly well. IIRC, [`pyflame`](https://github.com/uber-archive/pyflame) uses similar approach too.

parca and few more upcoming profilers use [eBPF](https://ebpf.io/). eBPF is a recent technology that allows to run userland code in the Kernel VM. It is a brilliant technology that is used in lots of areas like security, networking and observability for sure. I highly suggest to read some information on eBPF, it is definitely a massive topic that goes well beyond the scope of this blog post.

I think you get the idea: although there are various strategies to implement a sampling profiler under the hood, the underlying ideas/patterns stays same:
setup a periodic handler and read/aggregate stack trace data.

## Go CPU profiler sampler

Let's reiterate again: Go CPU profiler is a sampling profiler. In Linux, Go runtime uses `setitimer`/`timer_create/timer_settime` APIs to set up a `SIGPROF` signal handler. This handler is triggered at periodic intervals that is controlled by `runtime.SetCPUProfileRate` which is `100Mz(10ms)` by default. As a side note: surprisingly, there were some serious issues around the sampler of the Go CPU profiler until Go 1.18! You can see the gory details on the problems around it [here](https://www.datadoghq.com/blog/engineering/profiling-improvements-in-go-1-18/). As a brief summary, What I understand from the article is: `setitimer` API was the recommended way of triggering time based signals per-thread in Linux but it was not working as advertised. Please feel free to correct me if this claim is wrong. 

Let's see how you can enable the Go CPU profiler manually:

```go
if err := pprof.StartCPUProfile(f); err != nil {
    ...
}
defer pprof.StopCPUProfile()
```

Like mentioned, once `pprof.StartCPUProfile` is called, Go runtime enables the `SIGPROF` signal to be generated at specific intervals.

A simple visualization on how the Go CPU profiler sampler works:

![SIGPROF signal in the Go runtime](/sigprof.png)

