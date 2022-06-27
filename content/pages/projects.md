+++
title = "my projects"
path = "projects"
+++

I have written [a small CLI tool](https://github.com/videbar/kobo-highlights) to
manage the bookmarks from my ereader. It's built using Python, [Typer](
https://typer.tiangolo.com/) and [Rich](
https://rich.readthedocs.io/en/stable/introduction.html) and you can read more about
how it works on [my blogpost](
@/blog/buiding_a_python_cli_application_to_manage_my_bookmarks.md) about it.

In the past I worked at one of the laboratories of the European Commission where
[I wrote a report](https://doi.org/10.2760/997207) about the efficiency of electric
cars during V2G/V2H. This was a really cool project because I had to build a device
capable of connecting to the CAN of the car through the diagnostics port, requesting
battery information, decoding it and sending it to a database. I learnt a lot
about electronics, microcontrollers, and [the CAN bus standard](
https://en.wikipedia.org/wiki/CAN_bus).

During my Bachelor Thesis I worked in the automatic implementation of lightweight
cryptographic algorithms in HDLs. I reviewed most of the lightweight cryptographic
algorithms that were available at the time and experimented with a tool from Mathworks
called HDL coder. Ideally, it is capable of generating HDL code from Matlab code, and
the idea of my project was to test it on some of the lightweight cryptographic
algorithms. In the end the results were not that good (HDL coder could make the
conversion but it was far from efficient) but in the process I learnt about
cryptography theory, implemented a few real algorithms myself and learnt some
[verilog](https://es.wikipedia.org/wiki/Verilog).
