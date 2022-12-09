+++
title = "Implemeting Conway's Game of Life on a Microbit board using Rust "
date = 2022-12-09
+++
As part of my experience learning embedded programming in Rust, I implemented 
[Conway's Game of Life](https://en.wikipedia.org/wiki/Conway's_Game_of_Life) on a
[microbit-v2](https://tech.microbit.org/hardware/) board. This was a nice little project
that helped me get familiar with different aspects of the
[microbit board support crate](https://crates.io/crates/microbit) and more general
aspects of embedded programming such as concurrency and interrupts.

<!-- more -->

# The board

The Microbit board is an education device, designed to be useful when learning to code.
[It can be programmed in Scratch](https://scratch.mit.edu/microbit)
[or Python](https://python.microbit.org/v/3) and it includes among other peripherals,
a 5x5 LED grid, two buttons, a microphone, and a speaker.

Since it's designed for education, it's also the board used in the last edition of the
[Discovery book](https://docs.rust-embedded.org/discovery/microbit/), which makes it a
great device to start learning embedded Rust.

# This project

Roughly speaking, Conway's Game of Life is a way of simulating the evolution in time of
a population of simple life forms. Their universe is organized in a 2-dimensional grid
of square cells (think of a chess board) and each cell can either be populated (occupied
by a living being), or unpopulated (unoccupied). The game uses very simple rules to
emulate the behavior of living populations over time:

* Any populated cell with fewer than two populated neighbours dies, becoming unpopulated,
as if by underpopulation.

* Any populated cell with two or three populated neighbours lives on to the next
generation.

* Any live cell with more than three populated neighbours dies, as if by overpopulation.

* Any dead cell with exactly three live neighbours becomes a live cell, as if by
reproduction.

These rules are applied to the initial state of the game to generate the next state (or
the next generation), then they are applied to the second state to generate the third
one and so on. 

I should point out that, even though this system is called Conway's Game of Life, it's
a zero-player game, which means that once the system is set-up and the initial state
is chosen, there's nothing else for the user to do, but to watch as the population of
this tiny universe grows and evolves. If you're wondering why anyone would find this
interesting, it turns out that, even with such a simple set of rules, there's
[a lot of interesting patterns that can emerge](
https://en.wikipedia.org/wiki/Conway's_Game_of_Life#Examples_of_patterns), depending on
the initial state.

In my case, I'm interested in implementing the game on the Microbit board using the
5x5 LED grid. Off LEDs can represent unpopulated cells and on LEDs populated cells.
Even though technically Conway's Game of Life uses an infinite grid, 5x5 should be
enough to see some of the simpler patterns.

The game should advance to the next state periodically on its own, but I want to be
able to use one of the buttons of the Microbit to pause/unpause the evolution, and the
other one to jump to the next state immediately. This should provide a nice opportunity
to work with input pins and interrupts.

# Conway's Game of Life in Rust

The first step was to implement the game logic in Rust. I did this in a different module
using a `struct` that contains all the information about the current state of the
game. The grid is represented by a 5x5 matrix of booleans in which each boolean
indicates if the cell is populated (`true`) or unpopulated (`false`). The `struct` 
implements a method to update the state of the game, and a second method that exposes
the matrix using integers instead of booleans (`1` for `true` and `0` for `false`).
This is because the interface provided by the Microbit board support crate uses
integers to indicate which LEDs to turn on.

The most interesting detail about my implementation is related to how I count the
number of populated neighbouring cells in order to apply the update rules. You see, in
an infinite grid all cells are the same and have 8 neighbours, because of this, counting
the number of populated neighbours of any given cell is pretty straightforward.

In a finite grid, however there are cells on the border of the grid. As these border
cells have less than 8 neighbours, they need to be treated as special cases. This is not
hard to do, but it gets considerably messy. So what I did instead was take the 5x5 grid
and add two new rows, one above and one bellow, and two new columns, one to the left and
one to the right. The result is a 7x7 grid in which none of the cells from the original
5x5 grid are on the border.

For example, in the following image the red cell is in the corner of the 5x5 grid
and the yellow one is on the border. By adding the extra rows and columns, we end up
with a 7x7 grid in which both cells are in the interior, which means that they will
necessarily have 8 neighbours. By working with the "extended" grid, I don't have to
worry about special cases, since no cell from the original grid can be on the border
of the "extended" 7x7 grid.

{{
    image(src="/images/blog/rust_microbit_game_of_life/grid_expansion.svg",
       position="center")
}}

# The hardware game

The most basic embedded program has two parts, a setup that is executed once at the
beginning of the program, and an infinite loop that is executed forever while the device
is powered. This is implemented inside the `main()` function (the `!` on the signature
means that function never returns):

```rust
fn main() -> ! {

    // The setup goes here.

    loop {

        // The infinite loop goes here.

    }
}
```

In my first hardware implementation, I defined an initial state of the game on the
setup portion (together with all the necessary hardware setup) and, in the loop, I
included some code to calculate the next state and show it on the LEDs during
1.5 seconds. As expected, after flashing it to the board, I could see the game state
appear on the LED grid and update every 1.5 seconds.

This was simple enough, but I wanted to be able to pause/resume the game using
one of the buttons of the Microbit board. They way I implemented this was using a
GPIO interrupt. An interrupt is a function that is automatically called when some
pre-defined hardware event happens (in the case of the GPIO interrupt this event is
the press of a button). While the interrupt is running, the execution of the main
program is halted and, once the interrupt is done, the execution of the main program
resumes.

I defined a boolean variable to keep track of whether or not the game was
paused and used an interrupt to switch its value every time the button is pressed. The
game was then only updated it the boolean variable was set to `false`. Something to
consider, however, is how interrupts are called. They can't be called from `main()`,
instead they run on a different thread when they are triggered. This means that, in
order to have some shared variable, such as `PAUSED` between `main()` and
the interrupt I had to use [the special concurrency techniques for shared data](
https://docs.rust-embedded.org/book/concurrency/#global-mutable-data):

```rust
// The boolean is placed into a Mutex, so it can be shared across threads and into a
// RefCell to make it mutable through interior mutability.
static PAUSED: Mutex<RefCell<bool>> = Mutex::new(RefCell::new(false));

fn main() -> ! {

    // Setup the hardware and define the initial state of the game.

    loop {

        // The data inside the Mutex can only be accessed inside a critical section,
        // which is created with the free() function.
        cortex_m::interrupt::free(|cs| {

            // If the game is not paused, calculate the next state and display it
            // using the LEDs.

        }
    }
}

// This is how the interrupt is defined.
#[interrupt]
fn GPIOTE() {
    cortex_m::interrupt::free(|cs| {

        // Switch the value inside PAUSED.

    }
}
```

The [critical sections](
https://docs.rust-embedded.org/book/concurrency/#critical-sections) mentioned in the
example above are parts of the code which can not be halted by an interrupt. They are
used, among other things, when shared data is being accessed, or when the interrupts
are being configured.

The next thing I wanted to do was implementing a way of using the other button on the
Microbit to immediately update the game state if the game is paused. The board
support crate allows to detect button presses using the `GPIOTE` interrupt. When the
interrupt is called, it's possible to detect which of the two buttons was pressed
and run the appropriate code. In my case, if button A is pressed, I would switch the
value of `PAUSED` as in the previous example, but if button B is pressed and the game is
paused, it would call the `.next_state()` method that updates the state of the game.

The main thing to consider here is that the the state of the game now needs to be a
global variable since it's being accessed from `main()`, to update the state of the
game every 1.5 seconds, and from `GPIO()`, to perform the update when the game is
paused and button B is pressed. Because of this, it must also be placed inside a
`Mutex<RefCell<>>`:

```rust
static THE_GAME_IS_PAUSED: Mutex<RefCell<bool>> = Mutex::new(RefCell::new(false));
// The game state representation is a global mutable variable so it's placedl inside
// a Mutex and a RefCell. It's also placed inside an Option, to the initial value
// can be assigned later on, in main().
static GAME_STATE: Mutex<RefCell<Option<LifeState>>> = Mutex::new(RefCell::new(None));


fn main() -> ! {

    // Setup the hardware and define the initial state of the game.

    loop {
        // The critical section is now required to access both the boolean inside
        // THE_GAME_IS_PAUSED and the state of the game inside GAME_STATE.
        cortex_m::interrupt::free(|cs| {

            // If the game is not paused, calculate the next state and display it
            // using the leds.

        }
    }
}

// This is how the interrupt is defined.
#[interrupt]
fn GPIOTE() {
    cortex_m::interrupt::free(|cs| {

        // If button A was pressed, switch the value inside THE_GAME_IS_PAUSED.

        // If button B was pressed, update the state of the game inside GAME_STATE.

    }
}
```

# Using timers

The previous version works ok, but it has some problems related to the buttons
[bouncing](https://en.wikipedia.org/wiki/Switch#Contact_bounce). Button bouncing is a
physical phenomenon that makes it hard to detect when a switch or a button has been
pressed. It's caused by the physical properties of the components used to build the
button or switch. In my case, switch bouncing caused multiple calls to the `GPIOTE()`
interrupt on a single button press and erratic behavior in general.

There are certain ways to alleviate the bouncing problem; hardware "debouncers" can be
added to the electrical circuit, and there are certain algorithms that can be used to
debounce in software.

However, the easiest way to avoid switch bounce is by avoiding the `GPIOTE()` interrupt
altogether. Instead of trying the catch the exact moment in which the buttons are
pressed, I poll their states periodically. If a button was on the state "not pressed"
in the last check and now it's on the state "pressed", I now that it has been recently
pressed and I can then run the appropriate code.

This approach may seem sketchy, since I'm only checking the state of the buttons from
time to time, I may miss things that happen between checks. Also, even if a button is
being pressed, I'll have to wait until the next check to notice. In reality, none of
these are real problems as long as as the frequency with which the button states are
polled is high enough. For this implementation, I'm polling the buttons state every 6
milliseconds, which is enough to remove any bouncing and doesn't cause any appreciable
delay at a human scale.

The timer I'm using to poll the state of the switches is what's called a real time
counter (RTC). The
[nRF52833 processor](
https://infocenter.nordicsemi.com/index.jsp?topic=%2Fps_nrf52833%2Fkeyfeatures_html5.html)
that the Microbit board uses contains three RTCs, named RTC0, RTC1, and
RTC2. They can be used at the same time and the frequency of each one can be configured
independently. The RTCs can be set up in a way that an interrupt is called  on every
tick of the RTC clock. Here's an example of how I used RTC0 to check for the presses on
the button a:

```rust
// These are global mutable variables. It's necessary to have a variable that keeps
// track of the previous state of the button to detect button presses.
static BUTTON_A: Mutex<RefCell<Option<P0_14<Input<Floating>>>>> = Mutex::new(
    RefCell::new(None));
static BUTTON_A_WAS_PRESSED: Mutex<RefCell<bool>> = Mutex::new(RefCell::new(false));

fn main() -> ! {

    // Setup the hardware.

    loop {}
}

// This is interrupt is called every 6 ms.
#[interrupt]
fn RTC0() {
    cortex_m::interrupt::free(|cs| {

        // If button a is being pressed and BUTTON_A_WAS_PRESSED is false, a press just
        // occurred.

        // If button a is being pressed set BUTTON_A_WAS_PRESSED to true, otherwise set
        // it to false.

    }
}
```

For this second version, I also wanted to change how the LEDs were updated. The
Microbit crate offers two ways of controlling the 5x5 LED matrix on the board, the
[blocking display implementation](
https://docs.rs/microbit-v2/0.13.0/microbit/display/blocking/index.html), which is
simpler and easier to use, and the [non-blocking display implementation](
https://docs.rs/microbit-v2/0.13.0/microbit/display/nonblocking/index.html), which
requires more configuration but offers a higher degree of control over the LED matrix.
Previously I was using the blocking implementation, which is enough for this simple
use case, but since I was using a timer interrupt to poll the state of the buttons, I
wanted to try to use another one to automatically update the state of the game being
shown every second, which can be achieved using the non-blocking implementation.

In order to do so, it is necessary to configure a timer to drive the display (the
display refers to the 5x5 LED matrix). This timer will manage the refresh rate of the
display, once it's set up, images can be sent to the display to be shown. In my case,
I want to generate the next game state every second and send it to the display. I can
do this using another RTC, in this case RTC1, like this:

```rust
// The game state needs to be a global mutable variable.
static GAME_STATE: Mutex<RefCell<Option<LifeState>>> = Mutex::new(RefCell::new(None));

fn main() -> ! {

    // Setup the hardware and define the initial state of the game.

    loop {}
}

#[interrupt]
fn TIMER0() {

    // Drive the display.

}

// This is interrupt is called every second.
#[interrupt]
fn RTC1() {
    cortex_m::interrupt::free(|cs| {
        
        // Compute the next state of the game and show it on the display.
        
    }
}
```


As I've explained, the RTCs can be setup so an interrupt is called on every tick of the
clock, this is called a `Tick Interrupt`. One thing to consider is that the minimum
frequency at which the RTCs can run is 8 Hz. This means that a `Tick Interrupt` is
called, at a minimum, 8 times per second.

Since I want the game state to be updated every second, I couldn't use a
`Tick Interrupt`, instead I used [another type of RTC interrupt](
https://docs.rs/microbit-v2/0.13.0/microbit/hal/rtc/enum.RtcInterrupt.html) called
`Compare Interrupt`. A RTC has (as the name suggests) an internal counter. This counter
is increased by one on every tick of the clock. A `Compare Interrupt` can be configured
so that it's called when the counter reaches a certain number. If I set RTC1 to run at 8
Hz and I configure the `Compare Interrupt` to be called when the counter reaches 8, the
interrupt will be called after one second. If I then clear the counter when the
interrupt is called, I have a function that is called every second. This is the
configuration I used for the `RTC1()` interrupt on the previous example.

By combining this non-blocking implementation of the display with the previous
technique to poll the state of the buttons, it's possible to build a complete
implementation of the game of life program:

```rust
// For each button two global variables are required, one to hold the button struct
// and another one to keep track of its previous state.
static BUTTON_A: Mutex<RefCell<Option<P0_14<Input<Floating>>>>> = Mutex::new(
    RefCell::new(None));
static BUTTON_A_WAS_PRESSED: Mutex<RefCell<bool>> = Mutex::new(RefCell::new(false));

static BUTTON_B: Mutex<RefCell<Option<P0_23<Input<Floating>>>>> = Mutex::new(
    RefCell::new(None));
static BUTTON_B_WAS_PRESSED: Mutex<RefCell<bool>> = Mutex::new(RefCell::new(false));

// The game state and a flag to keep track of wether or not the game is paused are also
// defined as global mutable variables.
static GAME_STATE: Mutex<RefCell<Option<LifeState>>> = Mutex::new(RefCell::new(None));
static PAUSED: Mutex<RefCell<bool>> = Mutex::new(RefCell::new(false));

fn main() -> ! {

    // Setup the hardware and define the initial state of the game.

    loop {}
}

#[interrupt]
fn TIMER0() {

    // Drive the display.

}

// This is interrupt is called every second to update the game state.
#[interrupt]
fn RTC1() {
    cortex_m::interrupt::free(|cs| {
        
        // Compute the next state of the game and show it on the display, if the game
        // is not paused.
        
    }
}

// This is interrupt is called every 6 ms to poll the buttons.
#[interrupt]
fn RTC0() {
    cortex_m::interrupt::free(|cs| {

        // If button a is being pressed and BUTTON_A_WAS_PRESSED is false, pause/resume
        // the game.

        // If button a is being pressed set BUTTON_A_WAS_PRESSED to true, otherwise set
        // it to false.

        // If button b is being pressed and BUTTON_A_WAS_PRESSED is false, and if the
        // game is paused, update the game state and show it on the display.

        // If button b is being pressed set BUTTON_B_WAS_PRESSED to true, otherwise set
        // it to false.

    }
}
```

One problem I encountered was that [the Microbit crate's Board](
https://docs.rs/microbit-v2/0.13.0/microbit/board/struct.Board.html) only implements
one of the RTCs (RTC0), even though the other two are available at [the PAC](
https://docs.rs/microbit-v2/0.13.0/microbit/hal/pac/index.html). Because of this, I had
to defined my own board struct with all the peripherals that I used in this project.

# Sump up

This small project was a great way of getting used to some of the aspects of embedded
programming in Rust, such as interrupts and concurrency.

The first approach clearly showed how switch bouncing can be an issue, and the second
version showed how using a timer to poll the state of the buttons offers more reliable
results. 

Because the resulting program relies so heavily on interrupts, I'd like to have a look
at [the RTIC framework](https://rtic.rs/1/book/en/preface.html) for a potential
future version.

The entire code for this project can be found [on my github](
https://github.com/videbar/The-Game-of-Life-in-embedded-Rust), since it was a learning
project, there's plenty of comments going in detail about the different aspects of the
implementation.
