In my [previous post]() I made an overview of what PMU (Performance Monitoring Unit) is and what is PMU counter (PMC). We learned that there are fixed and programmable PMCs inside each PMU. We explored basics of counting and sampling mechanisms and left off on the advanced techniques and features for sampling. But before going into the main topic, let's discuss one common thing for counting and sampling. It's called multiplexing.

### Multiplexing and scaling events

The topic of multiplexing between different events in runtime is covered pretty well [here](https://perf.wiki.kernel.org/index.php/Tutorial#multiplexing_and_scaling_events), so I decided to take most of the explanation from there.

If there are more events than counters, the kernel uses time multiplexing to give each event a chance to access the monitoring hardware. With multiplexing, an event is not measured all the time. At the end of the run, the tool scales the count based on total time enabled vs time running. The actual formula is:

```
final_count = raw_count * ( time_running / time_enabled )
```

For example, say during profiling we were able to measure counter that we are interested 5 times, each measurement interval lasted 100ms (`time_enabled`). The program executed time is 1s(`time_running`). Total number of events for this counter is 10000 (`raw_count`). So, the `final_count` will be equal to 20000.

This provides an estimate of what the count would have been, had the event been measured during the entire run. It is very important to understand that this is an estimate not an actual count. Depending on the workload, there will be blind spots which can introduce errors during scaling.

This pretty much explains how "general-exploration" analysis in VTune Amplifier is able to collect near 100 different events just in single execution of the programm. For callibrating purposes, profiling tools usually have thresholds for different counters to decide if we can trust the measured number of events, or if it is too low to rely on it.

The easiest algorithm for multiplexing events is to manage it in round-robin fashion. Therefore each event will eventually get a chance to run. If there are N counters, then up to the first N events on the round-robin list are programmed into the PMU. In certain situations it may be less than that because some events may not be measured together or they compete for the same counter. 

To avoid scaling, one can try to reduce the number of events to be not bigger than the amount of physical PMCs available.

### Runtime overhead of characterizing and profiling

On the topic of runtime overhead in counting and sampling modes there is a [very good paper](https://openlab-archive-phases-iv-v.web.cern.ch/sites/openlab-archive-phases-iv-v.web.cern.ch/files/technical_documents/TheOverheadOfProfilingUsingPMUhardwareCounters.pdf) written by A. Nowak and G. Bitzes. They measured profiling overhead on a Xeon-based machine with 48 logical cores in different configurations: with disabled/enabled Hyper Threading, running tasks on all/several/one cores and collecting 1/4/8/16 different metrics. In my interpretation there is almost no runtime overhead (1-2%%) in counting mode and in sampling mode when you don't multiplex between different counters (with the caveat that you don't make the sampling frequency very high). However, if you'll try to collect more counters than the physical PMU counters available, you'll get performance hit of about 5-15% depending on the number of counters you want to collect and sampling frequency.

### Interrupt- vs. event-based sampling

Remember from my [last article]() I showed the number of steps which profiling tool does in order to collect statictics for your application. We initialize the counter with some number and wait until it overflows. On counter overflow, the kernel records information, i.e., a sample, about the execution of the program. What gets recorded depends on the type of measurement, but the key information that is common in all samples is the instruction pointer, i.e. where was the program when it was interrupted.

Interrupt-based sampling introduces skids on modern processors. That means that the instruction pointer stored in each sample designates the place where the program was interrupted to process the PMU interrupt, not the place where the counter actually overflows, i.e., where it was at the end of the sampling period. In some case, the distance between those two points may be several dozen instructions or more if there were taken branches. 

Let's take a look at the example:
![](/img/posts/PEBS_LBR/Interrupt-base-sampling.png){: .center-image }
Let's assume that on retirement of `instr1` we have an overflow of the counter that samples "instruction retired" events. Because of latency in the microarchitecture between the generation of events and the generation of interrupts on overflow, it is sometimes difficult to generate an interrupt close to an event that caused it. So by the time the interrupt is generated our IP has gone further by a number of instructions. When we reconstruct register state in interrupt service routine, we have slightly inaccurate data.

[Taken from here](https://perf.wiki.kernel.org/index.php/Tutorial#Sampling_with_perf_record)

This is possible to mitigate by having the processor itself store the instruction pointer (along with other information) in a designated buffer in memory – no interrupts are issued for each sample and the instruction pointer is off only by a single instruction, at most. This needs to be supported by the hardware, and is typically available only for a subset of supported events – this capability is called Processor Event-Based Sampling (PEBS) on Intel processors. You can also see people call it Precise Event-Based Sampling, but according to Intel manuals, first word is "Processor" not "Precise". But it basically means the same thing.

![](/img/posts/PEBS_LBR/Event-base-sampling.png){: .center-image }

### Processor Event-Based Sampling (PEBS)

Why it is only one instruction away?

from CERN:
An important thing to note is the skid of the instruction pointer – by the time the interrupt is issued and caught, the instruction pointer is likely to have progressed and thus give a slightly inaccurate location of the code that triggered the event. This is possible to mitigate by having the processor itself store the instruction pointer (along with other information) in a designated buffer in memory – no interrupts are issued for each sample and the instruction pointer is off only by a single instruction, at most. This needs to be supported by the hardware, and is typically available only for a subset of supported events – this capability is called Precise Event-Based Sampling (PEBS) on Intel processors. The skid will generate a shadow, which will “hide” any occurring events. Such behaviour can be a particular problem in regular loops, where a recurring scenario could mask a large portion of events.

PEBS is used with perf by adding :p and :pp suffix to the event specifier record -e event:pp

Capture beginning from Brendan's article, then from SDM B.3.3

When a performance counter is configured for PEBS, a PEBS record is stored in the PEBS 
buffer in the DS save area after the counter overflow occurs. This record contains the architectural state of the 
processor (state of the 8 general purpose registers, EIP register, and EFLAGS register) at the next occurrence 
of the PEBS event that caused the counter to overflow

With PEBS, the format of the samples is mandated by the processor. Each samples contains the machine state of the processor at the time the counter overflowed. The precision of PEBS comes from the fact that the instruction pointer recorded in each sample is at most one instruction away from where the counter actually overflowed. The skid is mimized compared to regular interrupted instruction pointer. Another key advantage of PEBS is that is minimizes the overhead because the Linux kernel is only involved when the PEBS buffer fills up, i.e., there is no interrupt until a lot of samples are available.

### LBR

References:
  CERN - overhead of counting and sampling
  http://www.brendangregg.com/perf.html
  SDM v3 - chapters 18,19
  SDM v2 - instructions: RDPMC, RDTSC, RDTSCP, RDMSR, WRMSR
