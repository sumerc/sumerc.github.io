---
title: "Deep dive into Go CPU profiler"
date: 2022-08-26T20:03:25+03:00
draft: true
---

# intro

It is safe to say that Go is one of its kind when it comes to profiling: Go runtime 
includes powerful and opiniated(!) profilers inside the runtime. In other languages like
Ruby, Python or Node.js there are some basic profilers and few APIs for writing 
profilers. In Go, however, runtime is pretty generous about observability tools. If you want to
learn more about the types of profilers, tracing tools that Go runtime has , I highly recommend Felix Geisendörfer's
[The Busy Developer's Guide to Go Profiling, Tracing and Observability](https://github.com/DataDog/go-profiler-notes/blob/main/guide/README.md)

As a curious engineer, I love to dig deep on how things work at a low level and I have always wanted to teach 
myself how Go CPU profiler works under the hood. So I started teaching myself how it works and 
then it turned into sharing this journey in a blog post. There is no such time, I did not find myself learning new things
when I read Go runtime and this was no exception. 

# basics

There are two types of profilers:
 1. tracing : xxx
 2. sampling : xxx

Every sampling profiler consists two basic parts:
1. sampler
2. data collection

Sampler triggers a callback at certain intervals and this callback collects profiling data(usually a stack trace).Different
profilers use different strategies to trigger the sampling interval.

# few examples

Linux `perf` uses `PMU` counters to trigger the data collection callback. XXX: More info. And it uses `perf_event_open(PERF_SAMPLE_STACK_USER,...)`
to obtain stack trace information. The captured stack traces are written to userspace via mmap'd ring buffer...

[`pyspy`](xxx) and [`rbspy`](xxx) are famous sampling profilers for Python and Ruby. They both ran as external processes and they periodically
read the target application memory to capture the stack trace of running threads. In Linux, they use `process_vm_readv` and if I am not mistaken
this API pauses the target application for few miliseconds during memory read. Then they chase pointers inside the memory they read to find the
current running thread structure and stack trace information. As you might guess, this is a error prone and complex approach but works 
surprisingly well. IIRC, [`pyflame`](https://github.com/uber-archive/pyflame) uses similar approach too.

parca and few more upcoming?/recent profilers use eBPF.

Go CPU profiler is a sampling profiler. In Linux, Go runtime uses `setitimer`/`timer_create?` APIs to set up a callback that is triggered at specific intervals that is controlled
by SetCPUProfileRate() which is 100Mz (10ms) by default.
