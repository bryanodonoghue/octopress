---
layout: post
title: "Fixing a boiling hot Macbook Retina 2015"
date: 2017-07-23 12:43:58 +0100
comments: true
categories: macbook, retina, machine check exception, linux
---

Problem statement
=================
My main machine is a 2015 macbook pro retina on which I run Linux natively (not in a VM)

{% img /images/macbook_fix/macbook-retina-display-2015.jpg %}

It's a beautiful machine with a high-density pixel display. It's also an oven whenever you start to do some serious work (tm) on it.
What do I mean by serious work ?

- Compiling clang
- Compiling Android
- Compiling a meaty Linux kernel

In tandem with say having a Google Hangouts call, watching youtube or streaming some music over whatever music streaming service it us you use. [offtopic:] I spotify BTW.

Background
==========

Q: In the first instance - what type of processor is in this MacBook ?
{% blockquote %}
cat /proc/cpuinfo
processor	: 7
vendor_id	: GenuineIntel
cpu family	: 6
model		: 70
model name	: Intel(R) Core(TM) i7-4960HQ CPU @ 2.60GHz
stepping	: 1
microcode	: 0xf
cpu MHz		: 3403.257
cache size	: 6144 KB
physical id	: 0
siblings	: 8
core id		: 3
cpu cores	: 4
apicid		: 7
initial apicid	: 7
fpu		: yes
fpu_exception	: yes
cpuid level	: 13
wp		: yes
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc aperfmperf eagerfpu pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm epb tpr_shadow vnmi flexpriority ept vpid fsgsbase tsc_adjust bmi1 hle avx2 smep bmi2 erms invpcid rtm xsaveopt dtherm ida arat pln pts
bugs		:
bogomips	: 5188.50
clflush size	: 64
cache_alignment	: 64
address sizes	: 39 bits physical, 48 bits virtual
power management:

{% endblockquote %}

So it's a <a href="http://ark.intel.com/products/76088/Intel-Core-i7-4960HQ-Processor-6M-Cache-up-to-3_80-GHz">Intel(R) Core(TM) i7-4960HQ CPU @ 2.60GHz</a> with four cores and eight threads.

Looking at the output of the 'sensors' application in a quiescent system we can see the following.

{% blockquote %}
$ sensors
BAT0-virtual-0
Adapter: Virtual device
temp1:        +33.7°C  

coretemp-isa-0000
Adapter: ISA adapter
Physical id 0:  +60.0°C  (high = +84.0°C, crit = +100.0°C)
Core 0:         +59.0°C  (high = +84.0°C, crit = +100.0°C)
Core 1:         +60.0°C  (high = +84.0°C, crit = +100.0°C)
Core 2:         +59.0°C  (high = +84.0°C, crit = +100.0°C)
Core 3:         +58.0°C  (high = +84.0°C, crit = +100.0°C)

applesmc-isa-0300
Adapter: ISA adapter
Left side  : 2158 RPM  (min = 2160 RPM, max = 6156 RPM)
Right side : 2001 RPM  (min = 2000 RPM, max = 5700 RPM)

{% endblockquote %}

- Four 'Core #' sensors reporting a temperature
- One 'Physical id 0' reporting a temperature
- A listing for a Left side fan and a Right side fan

In the default quiescent state - which for the purposes of this blog is an Ubuntu system booted to a desktop with two terminal windows open we see:

- Physical id 0 @ 55 c
- Core 0 @ 52
- Core 1 @ 53
- Core 2 @ 53
- Core 3 @ 54
- Left-Fan @ 2165/6156 RPM
- Right side @ 1999/5700 RPM

{% img /images/macbook_fix/quiescent_system_sensors_pre.png %}

Note there are four 'Core' sensors one for each of the cores previously identified for the 4960HQ and one 'Physical id 0' - physical id refers to the 'package' i.e. the four cores sit on one physical chip that gets soldered down to the board and this chip has its own individual sensor.

Take a look at the inside of the mac - the thing marked "CPU package" - that is what 'Physical id 0' referes to. The individual Cores are inside of that package.

Lets look inside the Macbook for starters.

{% img /images/macbook_fix/pcb_annotated.png %}

So here we can see two big fans "Left fan" (Left side) and "Right fan" (Right side), Air vents right beside those fans, a GPU (graphics processing unit) and CPU package (remember we said four cores inside of one 'package') plus a heat-strip to dissipate the heat from the CPU and GPU.

Now that strip is really not much to draw away any excess heat generated especially when you consider this is how the 4950HQ (a desktop cousin of the 4960HQ) is tooled up for cooling.

{% img /images/macbook_fix/4950hq-board.jpg %}

How bad is it really ?
======================
Are you just <a href="http://www.dictionary.com/browse/whinging">whinging</a> - how bad can it really be ?

Let's run a test.
Initial conditions are

- A freshly booted ubuntu system
- Two windows open one to build Linux one to monitor 'sensors'
- A running Webcam capture (to crank up the GPU)

What will we monitor ?
For the purposes of this test we will monitor the CPU temperature and the kernel log.

What will the test be ?
Against Linux commit 520eccdfe187 ("Linux 4.13-rc2") we will run this command with a Webcam stream active

```C
make clean; make mrproper; make distclean; make allyesconfig && time make bzImage -j 14
```

This will switch all kernel options for an x86_64 system to 'y' and then run a build of up-to 14 concurrent build processes.

To begin with the build seems OK recall the initial conditions

- Core 0 @ 52
- Core 1 @ 53
- Core 2 @ 53
- Core 3 @ 54
- Left-Fan @ 2165/6156 RPM
- Right side @ 1999/5700 RPM

{% img /images/macbook_fix/quiescent_system_sensors_pre.png %}

Viewed against our mid-build metrics

- Core 0 @ 75 C
- Core 1 @ 76 C
- Core 2 @ 76 C
- Core 3 @ 80 C
- Left-Fan @ 6096/6156 RPM
- Right side @ 5652/5700 RPM

{% img /images/macbook_fix/unfix_build_midway.png %}

Here we see the Core temperature is hot yes but below the 'high' threshold and way below critical.
The fans are working at the top of their limit to keep the processor in that state.

However the fans keep spinning and eventually we get this kernel message indicating the temperature sensors have detected a prolonged 'critical' temperature.

{% img /images/macbook_fix/overheat_machine_check.png %}

This is bad, very bad. Intel rates most of its processors at 105 degrees celcius maximum i.e. if your processor gets this hot a component inside of it will drive an exception to shut down the system entirely before things start to melt. In this case we can see that the ACPI descriptors provided by the BIOS have told Linux that 100 degrees celcius is the maximum.

In any case - it would be bad to allow things to continue on this way - IT WOULD BE BAD.

<iframe width="420" height="315" src="http://www.youtube.com/embed/jyaLZHiJJnE" frameborder="0" allowfullscreen></iframe>

What the hell can we do about it ?
==================================

Helpfully I'm not the first person to loose the rag with the over-heating Macs.
Somebody else has already taken the extreme step of drilling their <a href="https://www.macissues.com/2014/12/29/radical-fix-drill-holes-in-your-mac-to-make-it-run-cooler/">Mac</a>

I should say - my Dad did the marking and drilling here - since he has more experience/skill at this type of thing than I do :)

The first thing to do is to mark out where the drill holes will go.

{% img /images/macbook_fix/first_markout.png %}

Then drill the holes

{% img /images/macbook_fix/first_mesh.png %}

Then the second set of holes - this time marked in a box shape

{% img /images/macbook_fix/second_markout.png %}

And finally the finished product

{% img /images/macbook_fix/finished.png %}

Results
=======

So that's nice - you've drilled a hole in your €2,500 macbook - congratulations.
Does it make a difference to temperature and Machine-check-exceptions ?

In short - yes; whereas before the holes were drilled @ around 5900 RPM Left Side - the fan couldn't spin faster to evacuate out more air - so the processor just overheated.

{% img /images/macbook_fix/unfix_build_midway.png %}

Running the same test again we find that the Machine check exceptions have gone.

The hottest I've seen the processor get is 91 C - hot but still 9 degrees off max. Note how for a similar fan RPM we can operate at a much higher temperature.

{% img /images/macbook_fix/hottest.png %}

Now with more airflow - the fans still aren't maxed out when the temperature rises to 90+ degrees, and can allow the processor to draw more power - and rotate even faster to compensate.

Thus far - despite doing a few builds while watching some youtube, I haven't encountered the same error again.

It looks like

{% img /images/macbook_fix/tappy.jpg %}
