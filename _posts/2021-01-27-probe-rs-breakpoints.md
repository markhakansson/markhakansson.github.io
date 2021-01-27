---
layout: post
title:  "Working with ARM breakpoints in probe-rs"
date:   2021-01-27
categories: rust probe-rs debug arm 
---
If you've been working or tinkering with Embedded Rust, you might have stumbled upon the [probe-rs](https://probe.rs/) debugging toolkit in one way or another.
It is used by tools such as `cargo-embed`, `cargo-flash` and `probe-run`. It aims to replace the traditional GDB/OpenOCD stack and can be interchanged with it
for certain use cases already.

When working with breakpoints in an embedded Rust application. You might write your breakpoints in the program like
the following [RTIC](https://rtic.rs/) task
{% highlight rust %}
[#task]
fn task(cx: task::Context) {
    /* Some work here... */
    asm::bkpt();
}
{% endhighlight %}
When debugging with GDB it should usually work flawlessly. You will see that the program halts at that instruction and you will be able to
step and continue the program for there on. Using `probe-rs` instead you might have noticed that the program does not continue when trying
to step or continue running the program using the built in functions such as `Core::run()` or `Core::step()`. 

As for now it seems that there is no support to do so yet, using those functions. But fear not, the fix is quite easy!
What you need to do is manually increment the program counter to the next instruction and then continue the program.

{% highlight rust %}
    /* ...some code that takes the session and gives us the core */

    // Get the current address of the program counter
    let pc = core.registers().program_counter();
    let pc_val = core.read_core_reg(pc).unwrap();
    // Then increment the current position with 2 steps
    let step_pc = pc_val + 0x2;
    // Then overwrite the program counter with the previous value
    core.write_core_reg(pc.into(), step_pc).unwrap();
    core.step().unwrap();
{% endhighlight %}

The reason you need to increment with 2 steps is that an ARM instruction is usually 16-bits.
If you're working on ARM Thumb you will just need to increment with 1 step.

Now after doing this your program will step to the next instruction and you are no longer stuck at the breakpoint!

