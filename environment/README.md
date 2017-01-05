From [https://tip.golang.org/pkg/runtime/](https://tip.golang.org/pkg/runtime/).  
Last Modified: 2017-01-05

Environment Variables
=====================

The following environment variables (`$name` or `%name%`, depending on the host operating system) control the run-time behavior of Go programs.  
The meanings and use may change from release to release.

The `GOGC` variable sets the initial garbage collection target percentage.  
A collection is triggered when the ratio of freshly allocated data to live data remaining after the previous collection reaches this percentage.

The default is `GOGC=100`.  
Setting `GOGC=off` disables the garbage collector entirely.  
The runtime/debug package's `SetGCPercent` function allows changing this percentage at run time.  
See https://golang.org/pkg/runtime/debug/#SetGCPercent.

The `GODEBUG` variable controls debugging variables within the runtime.  
It is a comma-separated list of `name=val` pairs setting these named variables:

allocfreetrace:
---------------

```sh
# Disable
GODEBUG=allocfreetrace=0

# Enable
GODEBUG=allocfreetrace=1
```

`allocfreetrace=N`

setting `allocfreetrace=1` causes every allocation to be profiled and a stack trace printed on  
each object's allocation and free.

cgocheck:
---------

`cgocheck=N` 

setting `cgocheck=0` disables all checks for packages using cgo to incorrectly pass Go pointers to non-Go code.

Setting `cgocheck=1` (the default) enables relatively cheap checks that may miss some errors.

Setting `cgocheck=2` enables expensive checks that should not miss any errors, but will cause your program to run slower.

efence:
-------

`efence=N`

setting `efence=1` causes the allocator to run in a mode where each object is allocated on a  
unique page and addresses are never recycled.

gccheckmark:
------------

`gccheckmark=N`

setting `gccheckmark=1` enables verification of the garbage collector's concurrent mark phase by performing a  
second mark pass while the world is stopped.

If the second pass finds a reachable object that was not found by concurrent mark, the garbage collector will panic.

gcpacertrace:
-------------

```sh
# Disable
GODEBUG=gcpacertrace=0

# Enable
GODEBUG=gcpacertrace=1
```

setting `gcpacertrace=1` causes the garbage collector to print information about the internal state of the concurrent pacer.

gcshrinkstackoff:
-----------------

setting `gcshrinkstackoff=1` disables moving goroutines onto smaller stacks.

In this mode, a goroutine's stack can only grow.

gcstackbarrieroff:
------------------

setting `gcstackbarrieroff=1` disables the use of stack barriers that allow the garbage collector

to avoid repeating a stack scan during the mark termination phase.

gcstackbarrierall:
------------------

setting `gcstackbarrierall=1` installs stack barriers in every stack frame, rather than in exponentially-spaced frames.

gcrescanstacks:
---------------

setting `gcrescanstacks=1` enables stack re-scanning during the STW mark termination phase.  
This is helpful for debugging if objects are being prematurely garbage collected.

gcstoptheworld:
---------------

setting `gcstoptheworld=1` disables concurrent garbage collection, making every garbage collection a stop-the-world event.  

Setting `gcstoptheworld=2` also disables concurrent sweeping after the garbage collection finishes.

gctrace:
--------

setting `gctrace=1` causes the garbage collector to emit a single line to standards  
error at each collection, summarizing the amount of memory collected and the length of the pause.  

Setting `gctrace=2` emits the same summary but also repeats each collection. The format of this line is subject to change. Currently, it is:

```
gc # @#s #%: #+#+# ms clock, #+#/#/#+# ms cpu, #->#-># MB, # MB goal, # P
```

===

where the fields are as follows:

| symbol     | description                                        |
|------------|----------------------------------------------------|
| gc #       | the GC number, incremented at each GC              |
| @#s        | time in seconds since program start                |
| #%         | percentage of time spent in GC since program start |
| #+...+#    | wall-clock/CPU times for the phases of the GC      |
| #->#-># MB | heap size at GC start, at GC end, and live heap    |
| # MB goal  | goal heap size                                     |
| # P        | number of processors used                          |

The phases are stop-the-world (STW) sweep termination, concurrent mark and scan, and STW mark termination.  
The CPU times for mark/scan are broken down in to assist time (GC performed in line with allocation),  
background GC time, and idle GC time.  
If the line ends with "(forced)", this GC was forced by a `runtime.GC()` call and all phases are STW.

Setting `gctrace` to any `value > 0` also causes the garbage collector to emit a summary when memory  
is released back to the system.  

This process of returning memory to the system is called scavenging.  
The format of this summary is subject to change.

===

Currently it is:

| symbol | description                                          |
|--------|------------------------------------------------------|
| scvg#  | # MB released printed only if non-zero               |
| scvg#  | inuse: # idle: # sys: # released: # consumed: # (MB) |

where the fields are as follows:

| symbol      | description                                             |
|-------------|---------------------------------------------------------|
| scvg#       | the scavenge cycle number, incremented at each scavenge |
| inuse: #    | MB used or partially used spans                         |
| idle: #     | MB spans pending scavenging                             |
| sys: #      | MB mapped from the system                               |
| released: # | MB released to the system                               |
| consumed: # | MB allocated from the system                            |

memprofilerate:
---------------

setting `memprofilerate=X` will update the value of `runtime.MemProfileRate`.  
When set to `0` memory profiling is disabled. Refer to the description of  
MemProfileRate for the default value.

invalidptr:
-----------

defaults to `invalidptr=1`, causing the garbage collector and stack  
copier to crash the program if an invalid pointer value (for example, 1)  
is found in a pointer-typed location.  
Setting `invalidptr=0` disables this check. This should only be used as a temporary workaround  
to diagnose buggy code.  
The real fix is to not store integers in pointer-typed locations.

sbrk:
-----

setting `sbrk=1` replaces the memory allocator and garbage collector  
with a trivial allocator that obtains memory from the operating system and  
never reclaims any memory.

scavenge:
---------

`scavenge=1` enables debugging mode of heap scavenger.

scheddetail:
------------

setting `schedtrace=X` and `scheddetail=1` causes the scheduler to emit  
detailed multiline info every `X` milliseconds, describing state of the scheduler,  
processors, threads and goroutines.

schedtrace:
-----------

setting `schedtrace=X` causes the scheduler to emit a single line to standard  
error every `X` milliseconds, summarizing the scheduler state.

===

The net and net/http packages also refer to debugging variables in `GODEBUG`. See the documentation for those  
packages for details.

The `GOMAXPROCS` variable limits the number of operating system threads that can execute user-level Go  
code simultaneously.  
There is no limit to the number of threads that can be blocked in system calls on behalf of Go code;  
those do not count against the `GOMAXPROCS` limit.  
This package's `GOMAXPROCS` function queries and changes the limit.

The `GOTRACEBACK` variable controls the amount of output generated when a Go program fails due to an  
unrecovered panic or an unexpected runtime condition.  
By default, a failure prints a stack trace for the current goroutine, eliding functions internal to  
the run-time system, and then exits with exit code 2.  
The failure prints stack traces for all goroutines if there is no current goroutine or the failure is  
internal to the run-time.

-	`GOTRACEBACK=none`
	-	omits the goroutine stack traces entirely.  
-	`GOTRACEBACK=single` (the default)
	-	behaves as described above.  
-	`GOTRACEBACK=all`
	-	adds stack traces for all user-created goroutines.  
-	`GOTRACEBACK=system`
	-	like “all” but adds stack frames for run-time functions and shows goroutines created internally by the run-time.  
-	`GOTRACEBACK=crash`
	-	like “system” but crashes in an operating system-specific manner instead of exiting.  

For example, on Unix systems, the crash raises SIGABRT to trigger a core dump. For historical reasons,  
the `GOTRACEBACK` settings 0, 1, and 2 are synonyms for none, all, and system, respectively.  
The runtime/debug package's SetTraceback function allows increasing the amount of output at run time,  
but it cannot reduce the amount below that specified by the environment variable.  
See https://golang.org/pkg/runtime/debug/#SetTraceback.

The `GOARCH`, `GOOS`, `GOPATH`, and `GOROOT` environment variables complete the set of Go environment variables.  
They influence the building of Go programs (see https://golang.org/cmd/go and https://golang.org/pkg/go/build).

`GOARCH`, `GOOS`, and `GOROOT` are recorded at compile time and made available by constants or functions in this package,  
but they do not influence the execution of the run-time system.
