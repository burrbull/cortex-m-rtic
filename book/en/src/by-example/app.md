# The `app` attribute

This is the smallest possible RTIC application:

``` rust
{{#include ../../../../examples/smallest.rs}}
```

All RTIC applications use the [`app`] attribute (`#[app(..)]`). This attribute
must be applied to a `mod`-item. The `app` attribute has a mandatory `device`
argument that takes a *path* as a value. This must be a full path pointing to a
*peripheral access crate* (PAC) generated using [`svd2rust`] **v0.14.x** or
newer. More details can be found in the [Starting a new project](./new.md)
section.

The `app` attribute will expand into a suitable entry point so it's not required
to use the [`cortex_m_rt::entry`] attribute.

[`app`]: ../../../api/cortex_m_rtic_macros/attr.app.html
[`svd2rust`]: https://crates.io/crates/svd2rust
[`cortex_m_rt::entry`]: ../../../api/cortex_m_rt_macros/attr.entry.html

## `init`

Within the `app` module the attribute expects to find an initialization
function marked with the `init` attribute. This function must have
signature `fn(init::Context) -> (init::LateResources, init::Monotonics)`.

This initialization function will be the first part of the application to run.
The `init` function will run *with interrupts disabled* and has exclusive access
to Cortex-M where the `bare_metal::CriticalSection` token is available as `cs`.
And optionally, device specific peripherals through the `core` and `device` fields
of `init::Context`.

`static mut` variables declared at the beginning of `init` will be transformed
into `&'static mut` references that are safe to access. Notice, this feature may be deprecated in next release, see `task_local` resources.

[`rtic::Peripherals`]: ../../api/rtic/struct.Peripherals.html

The example below shows the types of the `core`, `device` and `cs` fields, and
showcases safe access to a `static mut` variable. The `device` field is only
available when the `peripherals` argument is set to `true` (default). In the rare case you want to implement an ultra-slim application you can explicitly set `peripherals` to `false`.

``` rust
{{#include ../../../../examples/init.rs}}
```

Running the example will print `init` to the console and then exit the QEMU
process.

```  console
$ cargo run --example init
{{#include ../../../../ci/expected/init.run}}
```

> **NOTE**: Remember to specify your chosen target device by passing a target
> triple to cargo (e.g `cargo run --example init --target thumbv7m-none-eabi`) or
> configure a device to be used by default when building the examples in `.cargo/config.toml`.
> In this case, we use a Cortex M3 emulated in QEMU so the target is `thumbv7m-none-eabi`.
> See [`Starting a new project`](./new.md) for more info.

## `idle`

A function marked with the `idle` attribute can optionally appear in the
module. This function is used as the special *idle task* and must have
signature `fn(idle::Context) - > !`.

When present, the runtime will execute the `idle` task after `init`. Unlike
`init`, `idle` will run *with interrupts enabled* and it's not allowed to return
so it must run forever.

When no `idle` function is declared, the runtime sets the [SLEEPONEXIT] bit and
then sends the microcontroller to sleep after running `init`.

[SLEEPONEXIT]: https://developer.arm.com/docs/100737/0100/power-management/sleep-mode/sleep-on-exit-bit

Like in `init`, `static mut` variables will be transformed into `&'static mut`
references that are safe to access. Notice, this feature may be deprecated in the next release, see `task_local` resources.

The example below shows that `idle` runs after `init`.

**Note:** The `loop {}` in idle cannot be empty as this will crash the microcontroller due to
LLVM compiling empty loops to an `UDF` instruction in release mode. To avoid UB, the loop needs to imply a "side-effect" by inserting an assembly instruction (e.g., `WFI`) or a `continue`.

``` rust
{{#include ../../../../examples/idle.rs}}
```

``` console
$ cargo run --example idle
{{#include ../../../../ci/expected/idle.run}}
```

## Hardware tasks

To declare interrupt handlers the framework provides a `#[task]` attribute that
can be attached to functions. This attribute takes a `binds` argument whose
value is the name of the interrupt to which the handler will be bound to; the
function adorned with this attribute becomes the interrupt handler. Within the
framework these type of tasks are referred to as *hardware* tasks, because they
start executing in reaction to a hardware event.

The example below demonstrates the use of the `#[task]` attribute to declare an
interrupt handler. Like in the case of `#[init]` and `#[idle]` local `static
mut` variables are safe to use within a hardware task.

``` rust
{{#include ../../../../examples/hardware.rs}}
```

``` console
$ cargo run --example hardware
{{#include ../../../../ci/expected/hardware.run}}
```

So far all the RTIC applications we have seen look no different than the
applications one can write using only the `cortex-m-rt` crate. From this point
we start introducing features unique to RTIC.

## Priorities

The static priority of each handler can be declared in the `task` attribute
using the `priority` argument. Tasks can have priorities in the range `1..=(1 <<
NVIC_PRIO_BITS)` where `NVIC_PRIO_BITS` is a constant defined in the `device`
crate. When the `priority` argument is omitted, the priority is assumed to be
`1`. The `idle` task has a non-configurable static priority of `0`, the lowest priority.

> A higher number means a higher priority in RTIC, which is the opposite from what
> Cortex-M does in the NVIC peripheral.
> Explicitly, this means that number `10` has a **higher** priority than number `9`.

When several tasks are ready to be executed the one with highest static
priority will be executed first. Task prioritization can be observed in the
following scenario: an interrupt signal arrives during the execution of a low
priority task; the signal puts the higher priority task in the pending state.
The difference in priority results in the higher priority task preempting the
lower priority one: the execution of the lower priority task is suspended and
the higher priority task is executed to completion. Once the higher priority
task has terminated the lower priority task is resumed.

The following example showcases the priority based scheduling of tasks.

``` rust
{{#include ../../../../examples/preempt.rs}}
```

``` console
$ cargo run --example preempt
{{#include ../../../../ci/expected/preempt.run}}
```

Note that the task `gpiob` does *not* preempt task `gpioc` because its priority
is the *same* as `gpioc`'s. However, once `gpioc` returns, the execution of
task `gpiob` is prioritized over `gpioa` due to its higher priority. `gpioa`
is resumed only after `gpiob` returns.

One more note about priorities: choosing a priority higher than what the device
supports (that is `1 << NVIC_PRIO_BITS`) will result in a compile error. Due to
limitations in the language, the error message is currently far from helpful: it
will say something along the lines of "evaluation of constant value failed" and
the span of the error will *not* point out to the problematic interrupt value --
we are sorry about this!
