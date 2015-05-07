---
layout: post
title: Measuring IRQ latency in RTAI – PCIe vs LPT
categories: rtos
tags: rtai fpga bluespec avalon pcie lpt
---

When designing a real-time system, it is of the utmost importance to ask:
How fast can the software react to an external event delivered by
the hardware components? How much this response time varies? Which is the worst
response time?

Despite the dependence on which computations need to be done by the real-time
software to solve the problem at hand, there is a lower bound on the response
time set by communication latency. External events are usually delivered
asynchronously, by the means of interrupt requests (IRQs). Therefore, it is
important to measure how long it takes for the real-time operating system to
answer IRQs.

We measure IRQ latency in a relatively modern computer running the
RTAI real-time OS. Despite having a recent motherboard, this computer still
possesses a parallel port (LPT) interface, which enables us to compare IRQ
delivery performance between PCI Express and LPT interfaces.

Our PCI Express interface is implemented by an Altera DE4 board, programmed with
an project designed employing the standard tool flow (Altera Qsys). This brings
us another interesting question – does the PCIe to Avalon bus abstraction
provided by Altera's tool flow impose significant overhead on the system?
Although we do not compare our design to any custom solution communicating
directly through the PCIe protocol, we are able to assess if IRQs are answered
in a reasonable time when compared to the traditional LPT interface.

<!--more-->

## Similar work

Measuring IRQ latency for a LPT interface in RTAI is an old and
well-known experiment:

  * [Captain.at](https://web.archive.org/web/20061207010914/http://www.captain.at/programming/rtai/parportint.php)
    had a very popular code example for measuring LPT IRQ latency using
    the `rt_get_time_ns` routine.

  * [Aldo Núñez](https://youtu.be/j3J7BNVkJjI?t=577) demonstrates how to
    reproduce the same kind of measurement employing an oscilloscope.

Here we employ both measurement techniques, take a statistically significant
amount of samples, and reproduce the experiments both for LPT and PCIe
interfaces.

# Materials

## Hardware

 * ASUSTeK P8P67-M PRO motherboard,
   Intel Core i3-2120 CPU,
   8 GB DDR3 1333MHz RAM
 * Altera [DE4-230](http://de4.terasic.com.tw) FPGA development board

<div class="text-center">
<img src="/postdata/2015-05-06/mb-setup.jpg" alt="Motherboard setup" style="width:90%" />
</div>

## Instruments

 * Tektronix TDS 460A oscilloscope

<div class="text-center">
<img src="/postdata/2015-05-06/tek-scope.jpg" alt="Tektronix TDS 460A" style="width:90%" />
</div>

## Software
 * Ubuntu 15.04
 * Linux kernel 3.4.55
 * RTAI [3.9.265.gd99c55e](http://linuxcnc.org/dists/wheezy/base/source/rtai_3.9.265.gd99c55e.tar.gz)<note>¹</note>

<note>
¹ We tried to use the latest release available at the moment (RTAI 4.1), but
it presented some issues with the RT timer. Periodic tasks simply did not
start to run and, depending on kernel configuration, the Linux clock was stopped
after calling <code>start_rt_timer</code> inside a RTAI module. Therefore we
resorted to using the same RTAI version as adopted by the Linux CNC project,
which we knew worked with this hardware. This is already the second machine on
which we observe this problem. We still need to report and help hunting
this bug.
</note>


# Methods

## IRQ echoer design

The LPT IRQ echoer consists simply on a parallel port connector with pin 2
(Data Out 0) connected to pin 10 (ACK). An IRQ is produced on the signal's
positive edge.

The PCIe IRQ echoer is implemented by the Bluespec SystemVerilog module
presented below. This module exposes two 32-bit addresses which are connected
to `bar0` in Qsys. When any value is written to the first address, an IRQ is
produced, and the value is held in an internal register. When the second
address is read, the value of this internal register is returned, and the IRQ
is cleared.

{%gist 841696cd2313d5a5ab6d %}

The IRQ flag is also exposed via an `irqflagtap` signal, which is configured as
a conduit in Qsys, allowing it to be routed to a GPIO pin and measured with an
oscilloscope. The figure below shows the connections of our Qsys system. See
[Altera's tutorial](ftp://ftp.altera.com/up/pub/Altera_Material/11.1/Tutorials/Using_PCIe_on_DE4.pdf)
to learn how to create the project from scratch in Quartus. The main difference
of our system from the tutorial's one is that we configured both `bar0` and
`bar2` as *32 bit Non-Prefetchable*.

<img src="/postdata/2015-05-06/pcie-qsys.png" style="width:100%;" />

## Measurement by oscilloscope

In LPT, the signal is measured directly from the ACK pin.
In PCIe, it is measured from the GPIO pin which exposes `irqflagtap`.

<div class="row">
  <div class="col-md-6 text-center">
    <label>LPT</label><br />
    <img src="/postdata/2015-05-06/probe-lpt.jpg" alt="LPT scope probe" height="400" />
  </div>
  <div class="col-md-6 text-center">
    <label>PCIe</label><br />
    <img src="/postdata/2015-05-06/probe-pcie.jpg" alt="PCIe scope probe"  height="400" /><br />
    <note>
    Just an adapter board for the DE4 GPIO header.
    The selected pin is connected directly via wires
    (not through the buffer IC).
    </note>
  </div>
</div>

## Measurement by software

A periodic real-time thread running at each $$100\mu\mathrm{s}$$ is implemented
by the `thread_main` function both in PCIe and LPT modules. It registers
the current time instant using `rt_get_time_ns` and then sends a signal to
the hardware asking for an IRQ to be sent. After the IRQ is dispatched by the
hardware, it is handled by an `irq_handler` real-time interrupt service routine.
As shown by the figures below, the first thing that this routine does (after
the conventional C function prologue) is measuring the current time instant.
These two measured time instants are subtracted from each other in order to
compute the latency.

<div class="row">
  <div class="col-md-6 text-center">
    <label>LPT</label><br />
    <img src="/postdata/2015-05-06/asm-lpt.png" alt="LPT IRQ routine" width="400" />
  </div>
  <div class="col-md-6 text-center">
    <label>PCIe</label><br />
    <img src="/postdata/2015-05-06/asm-pcie.png" alt="PCIe IRQ routine" width="400" /><br />
  </div>
</div>

## System load

During the measurements, I/O load was kept high by running

{% highlight bash %}
hping3 --icmp --fast otherhost
{% endhighlight %}

in a terminal, and the following in another one:

{% highlight bash %}
while :; do
    dd if=/dev/zero of=x bs=128M count=2
    sync; md5sum x; rm x; sync
done
{% endhighlight %}


# Results

## Measurement by oscilloscope

The oscilloscope was kept in the infinite persistence display mode, and data was
collected during 1 minute<note>²</note>.

<note>
² Intervals much longer than that caused the oscilloscope to freeze when saving
the screen hardcopy to a diskette.
</note>

<div class="row">
  <div class="col-md-6 text-center">
    <label>LPT</label><br />
    <img src="/postdata/2015-05-06/tek-lpt.svg" alt="Tektronix LPT" width="400" />
  </div>
  <div class="col-md-6 text-center">
    <label>PCIe</label><br />
    <img src="/postdata/2015-05-06/tek-pcie.svg" alt="Tektronix PCIe" width="400" /><br />
    <note>Sorry for the inductive noise</note>
  </div>
</div>

## Measurement by software

Data was collected during 14 minutes and plotted to a histogram. The vertical
dashed lines represent the minimum and maximum latency values observed for each
interface. Please note that the vertical axis has a logarithmic scale, therefore
it is not possible to distinguish between a single occurrence and no occurrences
at all.

<div class="text-center">
  <img src="/postdata/2015-05-06/latency-hist.svg" width="550" />
</div>


# Discussion

## Consistency between measurement techniques

### Timing diagram

<div class="text-center" style="padding:10px;">
  <img src="/postdata/2015-05-06/timing-diagram.svg" width="300" />
</div>

The timing diagram above explains why latency measurements done with the
oscilloscope ($$t_\mathrm{HW}$$) should be approximately equal to the ones done
by the real-time software ($$t_\mathrm{SW}$$). The software starts to count time
just before writing to the bus, but the signal is only observed by the external
hardware after a certain time $$t_\mathrm{rw}$$ (time to read/write to the bus).
However, after the software receives the IRQ and stops counting time,
approximately the same $$t_\mathrm{rw}$$ delay exists for the signal to stop
being observed by the external hardware.

### LPT

However, looking at the LPT interface results, an additional delay of about
$$2\mu\mathrm{s}$$ seems to exist in measurements done with the oscilloscope.
This is explained by the additional `inb` instruction which is used in the
`irq_handler` routine to verify if the IRQ has really been dispatched by the
parallel port (and not by another device which might be sharing the same IRQ).
This effectively increases to $$2 \times t_\mathrm{rw}$$ the time for the
oscilloscope to stop observing a digital high signal after the IRQ is delivered.

As in our particular system IRQ 5 (used by LPT) was not shared with other
devices (as confirmed by looking at `/proc/interrupts`), we did another
experiment after removing this check, by changing the IRQ handler to:

{% highlight c %}
static int irq_handler(unsigned irq, void *cookie_) {
    const RTIME diff = rt_get_time_ns() - timestamp_before_write;
    const uint8_t data = 0xFF; // was: inb(LPT);
    const int irq_is_ours = (data & 0x1) == 1;

    if (likely(irq_is_ours)) {
        rtf_put(FIFO_DT, (void*)&diff, sizeof(diff));
        outb(0x00, LPT);
    }

    rt_unmask_irq(irq);
    return irq_is_ours;
}
{% endhighlight %}

The figure below superposes measurements done using the oscilloscope with and
without the `inb` instruction included in the IRQ handler. In the
experiment using the routine without `inb`, the oscilloscope measurements are
very consistent with the software-measured histograms. We conclude that
$$t_\mathrm{rw} \approx 2\mu\mathrm{s}$$ for the LPT interface.

<div class="text-center" style="padding:10px;">
  <img src="/postdata/2015-05-06/tek-lpt-inb.svg" width="400" />
</div>

### PCIe

Contrary to the LPT interface, we designed our PCIe interface in such a way
that we can verify if the interrupt was really caused by our PCIe board and
clear the IRQ flag in a single operation. Whereas in LPT we first need to call
`inb` to check if the IRQ is ours then call `outb` to clear the relevant
data output bit, in our PCIe hardware a single read to the `0x04` register
completes both tasks.

## Latency and jitter results

If we consider minimum latency, mean latency and jitter, the PCIe interface
results were clearly superior to LPT. Whereas for LPT we measured
$$5.7\mu\mathrm{s}$$ minimum latency, $$8.1\mu\mathrm{s}$$ mean latency and
$$0.8\mu\mathrm{s}$$ standard deviation, PCIe presented, respectively,
$$4.8\mu\mathrm{s}$$, $$6.4\mu\mathrm{s}$$ and $$0.5\mu\mathrm{s}$$. However,
the software-measured data caught a particularly strange behavior which was not
visible in oscilloscope measurements.

A fraction of about $$10^{-5}$$ of the PCIe latency measurements behaved as
outliers, forming a secondary distribution of values larger than the observed
LPT latencies. Thus the PCIe interface was not better than LPT in the worst-case
timing, a behavior which was not expected by us. We were still not able to
trace back the reason, since the IRQ to which the PCIe board was attributed in
our system was not shared with other devices, thus there is not any immediately
obvious source of hardware resource contention.
