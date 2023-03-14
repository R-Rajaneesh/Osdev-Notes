# Timer

Timers are useful for all sorts of things, from keeping track of real-world time to forcing entry into the kernel to allow for pre-emptive multitasking. There are many timers available, some standard and some not.

In this chapter we're going to take a look at the main timers used on x86 and what they're useful for.

## Types and Characteristics

At a high level, there a few things we might want from a timer: 

- Can it generate interrupts? And does it support a periodic mode?
- Can we poll it to determine how much time has passed?
- Does the main clock count up or down?
- What kind of accuracy, precision and latency does it have?

At first most of these questions might seem unnecessary, as all we really need is a periodic timer to generate interrupts for the scheduler right? Well that can certainly work, but as you do more things with the timer you may want to accurately determine the length of time between two points of execution. This is hard to do with interrupts, and it's easier to do with polling. A periodic mode is also not always available, and sometimes you are stuck with a one-shot timer.

For x86, the common timers are:

- The PIT: it can generate interrupts (both periodic and one-shot) and is pollable. When active it counts down from a reload value to 0. It has a fixed frequency and is very useful for calibrating other timers we don't know the frequency of. However it does come with some latency due to operating over port IO, and it's frequency is low compared to the other timers available.
- The local APIC timer: it is also capable of generating interrupts (periodic and oneshot) and is pollable. It operates in a similar manner to the PIT where it counts down from a reload value towards 0. It's often low latency due to operating over MMIO and comes with the benefit of being per-core. This means each core can have it's own private timer, and more cores means more timers.
- The HPET: capable of polling with a massive 64-bit main counter, and can generate interrupts with a number of comparators. These comparators always support one-shot operation and may optionally support a periodic mode. It's main clock is count-up, and it is often cited as being high-latency. It's frequency can be determined by parsing an ACPI table, and thus it serves as a more accurate alternative to the PIT for calibrating other timers.
- The TSC: the timestamp-counter is tied to the core clock of the cpu, and increments once per cycle. It can be polled and has a count-up timer. It can also be used with the local APIC to generate interrupts, but only one-shot mode is available. It is often the most precise and accurate timer out of the above options.

We're going to focus on setting up the local APIC timer, and calibrating it with either the PIT or HPET. We'll also have a look at a how you could also use the TSC with the local APIC to generate interrupts.

## Programmable Interval Timer (PIT)

- legacy hardware, orginal PC timer. Is often emulated by newer hardware (the HPET) on newer platforms.
- Still useful because we know it's frequency ahead of timer.

The PIT is actually from the original IBM PC, and has remained as a standard device all these years. Of course these days we don't have a real PIT in our computers, rather the device is emulated by newer hardware that pretends to be the PIT until configured otherwise. Often this hardware is the HPET (see below).

On the original PC the PIT also had other uses, like providing the clock for the RAM and the oscillator for the speaker. Each of these functions is provided by a 'channel', with channel 0 being the timer (channels 1 and 2 are the other functions). On modern PITs it's likely that only channel 0 exists, so the other channels are best left untouched.

Despite it being so old the PIT is still useful because it provides several useful modes and a known frequency. This means we can use it to calibrate the other timers in our system, which we dont always know the frequency of.

The PIT itself provides several modes of operation, although we only really care about a few of them:

- Mode 0: provides a one-shot timer.
- Mode 2: provides a periodic timer.

To access we PIT we use a handful of IO ports:

| I/O Port | Description          |
|----------|----------------------|
|  0x40    | Channel 0 data Port  |
|  0x41    | Channel 1 data Port  |
|  0x42    | Channel 2 data Port  |
|  0x43    | Mode/Command register|

### Theory Of Operation

As mentioned the PIT runs at a fixed frequency of 1.19318MHz. This is an awkward number but it makes sense in the context of the original PC. The PIT contains a pair of registers per channel: the count and reload count. When the PIT is started the count register is set to value of the reload count, and then every time the main clock ticks (at 1.19318MHz) the count is decremented by 1. When the count register reaches 0 the PIT sends an interrupt. Depending on the mode the PIT may then set the count register to the reload register again (in mode 2 - periodic operation), or simple stay idle (mode 0 - one shot operation).

The PIT's counters are only 16-bits, this means that the PIT can't count up to 1 second. If you wish to have timers with a long duration like that, you will need some software assistance by chaining time-outs together.

### Example

As an example let's say we want the PIT to trigger an interrupt every 1ms (1ms = 1/1000 of a second). To figure out what to set the reload register to (how many cycles of the PIT's clock) we divide the clock rate by the duration we want:

```
1,193,180 (clock frequency) / 1000 (duration wanted) = 1193.18 (Hz for duration)
```

One problem is that we can't use floating point numbers for these counters so we truncate the result to 1193. This does introduce some error, and you can correct for this over a long time if you want. However for our purposes it's small enough to ignore, for now.

To actually program the PIT with this value is pretty straightfoward, we first send a configuration byte to the command port (0x43) and then the reload value to the channel port (0x40).

The configuration byte is actually a bitfield with the following layout:

| Bits   | Description                                                                                                          |
|--------|----------------------------------------------------------------------------------------------------------------------|
| 0      | Selects BCD/binary coded decimal (1) or binary (0) encoding. If you're unsure, leave this as zero. |
| 1 - 3  | Selects the mode to use for this channel. |
| 4 - 5  | Select the access mode for the channel: generally you want 0b11 which means we send the low byte, then the high byte of the 16-bit register. |
| 6 - 7  | Select the channel we want to use, we always want channel 0. |

For our example we're going to use binary encoding, mode 2 and channel 0 with the low byte/high byte access mode. This results in the following config byte: `0b00110100`.

Now it's a matter of sending the config byte and reload value to the PIT over the IO ports, like so:

```c
void set_pit_periodic(uint16_t count) {
    outb(0x43, 0b00110100);
    outb(0x40, count & 0xFF); //low-byte
    outb(0x40, count >> 8); //high-byte
}
```

Now we should be getting an interrupt from the PIT every millisecond! By default the PIT appears on irq0, which may be remapped to irq2 on modern (UEFI-based) systems. Also be aware that the PIT is system-wide device, and if you're using the APIC you will need to program the IO APIC to route the interrupt to one of the LAPICs.

## High Precision Event Timer (HPET)

The HPET was meant to be the successor to the PIT as a system-wide timer, with more options however it's design has been plagued with latency issues and occasional glitches. With all that said it's still a much more accurate and precise timer than the PIT, and provides more features. It's also worth noting the HPET is not available on every system, and can sometimes be disabled via firmware.

### Discovery

To determine if the HPET is available you'll need access to the ACPI tables. Handling these is covered in a separate chapter, but we're after one particular SDT with the signature of 'HPET'. If you're not familiar with ACPI tables yet, feel free to come back to the HPET later.

This SDT has the standard header, followed by the following fields:

```c
struct HpetSdt {
    ACPISDTHeader header;
    uint32_t event_timer_block_id;
    uint32_t reserved;
    uint64_t address;
    uint8_t id;
    uint16_t min_ticks;
    uint8_t page_protection;
}__attribute__((packed));
```

We're mainly interested in the `address` field which gives us the physical address of the HPET registers. The other fields are explained in the HPET specification but are not needed for our purposes right now.

As with any MMIO you will need to map this physical address into the virtual address space so we can access the registers with paging enabled.

### Theory Of Operation

The HPET consists of a single main counter (that counts up) with some global configuration, and a number of comparators that can trigger interrupts when certain conditions are met in relation to the main counter. The HPET will always have at least 3 comparators, but may have up to 32.

The HPET is similar to the PIT in that we are told the frequency of it's clock. Unlike the PIT, the HPET spec does not give us the frequency directly, we have to read it from the HPET registers. 

Each register is accessed by adding an offset to the base address we obtained before. The main registers we're interested in are:

- General capabilities: offset 0x0.
- General configuration: offset 0x10.
- Main counter value: 0xF0.

## Local APIC Timer
TODO: LAPIC timer, calibration using the above

## Timestamp Counter (TSC)
TODO: TSC how to, calibration

## Useful Timer Functions
TODO: polled_sleep()
TODO: arm_interrupt_timer()
//--- PREV CONTENT BELOW ---

## Why we need the calibration?

Because according to the intel ia32 documentation the APIC frequency is the same of the bus frequency or the core crystal frequency (divided by the chosen frequecny divider) so this basically depend on the hardware we are running. 

There are different ways to calibrate the apic timer, the one explained here is the most used and easier to understand. 

## IRQ 

The PIT timer is connected to the old PIC8259 IRQ0 pin, now if we are using the APIC, this line is connected to the Redirection Table entry number #2 (offset: 14h and 15h).

While the APIC timer irq is always using  the lapic LVT entry 0. 


## Steps for calibration

These are at a high level the steps that we need to do to calibrate the APIC timer: 

1. Configure the PIT Timer
2. Configure the APIC timer 
3. Reset the APIC counter
4. Wait some time that is measured using another timing source (in our case the PIT)
5. Compute the number of ticks from the APIC counter
6. Adjust it to the desired measure
7. Divide it by the divider chosen, use this value to raise an interrupt every x ticks
8. Mask the PIT Timer IRQ

### Configure the PIT Timer (1)

Refer to the paragraph above, what we want to achieve on this step is configure the Pit timer to generate an interrupt with a specific interval (in our example 1ms), and configure an IRQ handler for it. 

#### Configure the PIT Timer: IRQ Handling

The irq handling will be pretty easy, we just need to increment a global counter variable, that will count how many ms passed (the "how many" will depend on how you configured the pit), just keep in mind that this variable must be declared as volatile. The irq handling function will be as simple as it seems: 

```C
void irq_handler() {
    pit_ticks++;
}
```

### Configure the APIC Timer (2)

This step is just an initialization step for the apic, before proceeding make sure that the timer is stopped, to do that just write 0 to the Initial Count Register. 

The APIC Timer has 3 main register: 

* The Initial Count Register at Address 0xFEEO 0380 (this has to be loaded with the initial value of the timer counter, every time time the counter it reaches 0 it will generate an Interrupt
* The current Count Register at Address 0xFEE0 0390 (current value of the counter)
* The Divide Configuration register at 0xFEE0 03E0 The Timer divider.


In this step we need first to configure the timer registers: 

* The Initial Count register should start from the highest value possible. Since all the apic registers are 32 bit, the maximum value possible is: (0xFFFFFFFF)
* The Current Count register doesn't need to be set, it is set automatically when we initialize the initial count. This register basically is a countdown register.
* The divider configuration register it's up to us (i used 2)

### Wait some time (4)

In this step we just need to make an active waiting, basically something like: 

```c
while(pitTicks < TIME_WE_WANT_TO_WAIT) {
    // Do nothing...
}
```

pitTicks is a global variable set to be increased every time an IRQ from the PIT has been fired, and that depends on how the PIT is configured. So if we configured it to generate an IRQ every 1ms, and we want for example 15ms, will need the vlaue for `TIME_WE_WANT_TO_WAIT` will be 15. 

### Compute the number of ticks from the APIC Counter and obtain the calibrated value  (5,6,7)

That step is pretty easy, we need to read the APIC current count register (offset: 390). 

This register will tell us how many ticks are left before the register reaches 0. So to geth the number of ticks done so far we need to:

```c
apic_ticks_done = initial_count_reg_value - current_count_reg_value
````

this number tells us how many apic ticks were done in the `TIME_WE_WANT_TO_WAIT`. Now if we do the following division:

```c
apic_ticks_done / TIME_WE_WANT_TO_WAIT;
```

we will get the ticks done in unit of time. This value can be used for the initial count register as it is (or in multiple depending on what frequency we want the timer interrupt to be fired)

This is the last step of the calibration, we can start the apic timer immediately with the new calibrated value for the initial count, or do something else before. But at this point the PIT can be disabled, and no longer used. 


## Useful links

* [Ehtereality osdev Notes - Apic/Timing/Context Switching](https//ethv.net/workshops/osdev/notes/notes-4.html)
* [OSdev Wiki - Pit page](https://wiki.osdev.org/Programmable_Interval_Timer)
* [Brokern Thron Osdev Series](http://www.brokenthorn.com/Resources/OSDev16.html)