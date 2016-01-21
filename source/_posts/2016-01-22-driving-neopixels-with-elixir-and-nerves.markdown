---
layout: post
title: "Driving NeoPixels with Elixir and Nerves"
date: 2016-01-22 22:41:20 -0500
comments: true
categories: 
- Elixir
- Nerves
- NeoPixel
- Embedded
- Hardware
---

I'd been hearing good things about [Elixir][elixir], played with it a little on [Exercism][exercism], and even went to a local Elixir meetup a couple of times.
I wanted to use Elixir for something to give me a chance to really understand how it works and how it's different from my more familiar languages like Ruby.

Then I saw the talk ["Embedded Elixir in Action" by Garth Hitchens][ghitchens_talk] about using [Nerves][nerves] to develop real-world Elixir-based embedded systems.
I'd had an original [Raspberry Pi B][pi_b] sitting on a shelf for a few years, so this seemed like it would be a good opportunity to learn Elixir and finally use that hardware for something.
I also had a bunch of [WS2812B "NeoPixel" RGB LEDs][neopixel] that I was itching for an excuse to use for something.
(Caution: If you browse the available NeoPixel products on Adafruit, you may not be able to resist buying something).

At this point, I was unable to resist hitting these three birds with one stone!

[elixir]: http://elixir-lang.org
[exercism]: http://exercism.io
[ghitchens_talk]: https://www.youtube.com/watch?v=kpzQrFC55q4
[nerves]: http://nerves-project.org
[pi_b]: https://www.raspberrypi.org/products/model-b/
[neopixel]: https://www.adafruit.com/category/168

<!-- more -->

## A Brief Introduction

I started this project pretty new to Elixir, having just begun to read ["Programming Elixir" by Dave Thomas][programming_elixir].
I also wasn't familiar with embedded systems development beyond writing some hobbyist C code for AVR microsontrollers in the distant past.
I want to take an opportunity to chime in with the other voices in saying that you don't have to really understand all these things deeply in order to get started and build a useful project.
In this blog post, I will attempt to remember and describe the things that confused me along the way, to hopefully accelerate others' ability to get up to speed.

[programming_elixir]: https://pragprog.com/book/elixir/programming-elixir

### Elixir, Erlang, and Processes

Elixir is a functional programming language that runs on the Erlang virtual machine, called BEAM.
This is just like how the Java Virtual Machine (JVM) was designed for running Java code, but can also host code written in other languages, like Clojure or Ruby (via JRuby).
In both instances, a brand-new language (e.g. Elixir or Clojure) could be born with an already-robust, production-ready run-time environment istead of starting from scratch.

In addition to using a functional style, Elixir has a few other key concepts to understand when you're getting started.
In Elixir, the code is arranged as functions that are grouped into Modules, which normally run many Processes in order to accomplish their purpose.
These Processes in the Erlang VM are much more lightweight than an Operating System process, so it's not unusual to have many thousands of them running at any time, much as you might have many Objects instantiated in a Ruby-based system.
Also similar to Ruby's Objects, a Process in Elixir is how state is stored and accessed by sneding messages between Processes.
Speaking of state, it's also worth mentioning that the data structures in Elixir are all immutable.
When the state of a Process needs to be changed, it is accomplished by replacing its state with a new state rather than modifying parts of the state in-place.
This is confusing at first for a new Elixir programmer, but it quickly becomes natural and enables Elixir to do some great things under-the-hood to enable efficient concurrency and garbage-collection.

### OTP and Applications

OTP is a _really_ cool feature that Elixir inherits from its Erlang heritage.
One thing that wasn't obvious to me at first is that what a developer might normall call 'an application' is, in OTP terms, a collection on interacting OTP Applications.
For example, if you want to log things to `stdout` or `stderr`, you might include the `Logger` Application in your `mix.exs` file, like this:

``` elixir mix.exs
defmodule MyProject.Mixfile do
  use Mix.Project

  # ...

  def application, do: [
    applications: [:logger],
    mod: {MyProject, []}
  ]
end
```

Then, when you run your project, the `mod` option says that the `MyProject` module will start the supervision tree for my application.
The `applications` option says to also start additional OTP Application(s), their process(es) waiting to receive messages from other Processes.

Besides that, there is plenty more to learn about OTP, but you'll probably learn best by just reading up on it and trying it out for yourself.
[The official Elixir getting-started introduction to Mix and OTP][mix_otp] is unsurprisingly a great place to start.

Now, onward to the main point!

[mix_otp]: http://elixir-lang.org/getting-started/mix-otp/

## Getting Started with Nerves

Having established how fun and exciting Elixir is, let's talk about how to easily get Elixir code running on a Raspberry Pi (or similar Linux-based embedded development platform).
The tool to accomplish this is called [Nerves][nerves].
In a nutshell, Nerves wraps up the [Buildroot][buildroot] tool and most of the configuration settings needed to cross-compile a stripped-down Linux image for your target platform of choice.

As of this writing, there's a brand-new tool called [Bakeware][bakeware] that can be used to simplify the process I'm going to describe below for many common use-cases.
Wendy Smoak recently wrote a [blog post explaining how to use Bakeware][wsmoak_nerves_bake], so check that out if you're interested.
I'll explain the alternate method I've been using to build a images from scratch the old-fashioned way.

[nerves]: http://nerves-project.org
[buildroot]: https://buildroot.org
[bakeware]: http://www.bakeware.io
[wsmoak_nerves_bake]: http://wsmoak.net/2016/01/11/embedded-elixir-nerves-bake.html

### Setting up a Nerves Build Environment

Much has been written about how to get Nerves running on Mac OSX, so I'm going to contribute the process I used to get going on Windows using a Linux VM.
I'm using Ubuntu 14.04 since that's what is described on the Nerves website and I wanted to avoid figuring out which equivalent packages to use for CentOS.

From a fresh install of Ubuntu, you'll need to install the following packages to get started:

``` console bash
sudo apt-get install git g++ libssl-dev libncurses5-dev bc m4 make unzip
```

Then, check out the main Nerves repository:

``` console bash
git clone git@github.com:nerves-project/nerves-system-br.git
```

After that finishes, run `make help` to get a list of supported platforms:

``` console bash
Nerves System Help
------------------

Targets:
  all                           - Build the current configuration
  burn                          - Burn the most recent build to an SDCard (requires sudo)
  system                        - Build a system image for use with bake
  clean                         - Clean everything - run make xyz_defconfig after this

Configuration:
  menuconfig                    - Run Buildroot's menuconfig
  linux-menuconfig              - Run menuconfig on the Linux kernel

Nerves built-in configs:
  bbb_linux_defconfig           - Build for bbb_linux
  nerves_bbb_elixir_defconfig   - Build for nerves_bbb_elixir
  nerves_bbb_erlang_defconfig   - Build for nerves_bbb_erlang
  nerves_galileo_elixir_defconfig - Build for nerves_galileo_elixir
  nerves_rpi2_elixir_defconfig  - Build for nerves_rpi2_elixir
  nerves_rpi2_erlang_defconfig  - Build for nerves_rpi2_erlang
  nerves_rpi_elixir_defconfig   - Build for nerves_rpi_elixir
  nerves_rpi_erlang_defconfig   - Build for nerves_rpi_erlang
  nerves_rpi_lfe_defconfig      - Build for nerves_rpi_lfe
```

Since my target platform is an original Raspberry Pi Model B, I then ran

``` console bash
make nerves_rpi_elixir_defconfig
```

This sets up the configuration options to build a Linux image that will boot into Elixir's `iex` shell on the terminal on the Raspberry Pi's HDMI monitor output.
It's possible to re-configure the terminal to use the UART pins on the board, so that you could connect to it using PuTTY over an FTDI cable, so check out the how-to on [the `nerves-system-br` README][nerves_readme] if you want to do that.
Instead, I just plugged a monitor into the HDMI port and a USB keyboard into one of the USB ports.

Once the default configuration is done, just run `make` to build the default system image.
This takes a really long time, depending on how fast your computer/hard drive/Internet connection are.

While you're waiting for it to get done, maybe you could read about Bakeware, which lets you build your app using pre-baked system images instead of doing this build process yourself.
For most people getting started, that's probably a better way to go.

``` console bash
make
```

This will result in the base firmware image being written to `buildroot/output/images/nerves-rpi-base.img`.
At this point, I confirmed that things were working properly by copying this `.img` file to my Windows host machine using [WinSCP][winscp], then burned it to an SD card using [Win32DiskImager][w32di].

Yes, I downloaded and ran a Windows executable (As an Administrator!) from Sourceforge.
I feel bad about it, but apparently, that's just how people get the functionality of `dd` on Windows.
Let's be honest, it's about the same thing as a sudo-curl-bash installer we've come to accept in recent years.

[winscp]: https://winscp.net/
[w32di]: http://sourceforge.net/projects/win32diskimager

After the SD card is done being written, I put it in my Raspberry Pi and booted it up.
Lo and behold, (after only four seconds!) I was greeted by an `iex` prompt.

{% img center /images/posts/2016-01-22-driving-neopixels-with-elixir-and-nerves/nerves_hello_world.jpg Hello World from Elixir and Nerves on a Raspberry Pi %}

## Driving NeoPixels from a Raspberry Pi

With all that out of the way, we're ready for the actual blinkenlights.
Well, almost.

First, we need a way to drive the 5V NeoPixel data intput using the Raspberry Pi's 3.3V outputs.
One easy way to accomplish this is with a [74AHCT125 Level-Shifter chip][74ahct125], but I didn't have one laying around and didn't want to order one.
What I _did_ have laying around was an [SN74ALS1035N Non-Inverting Buffer][sn74als1035n].
The trick was that the inputs happen to accept a High-level input voltage of only 2V, while the outputs are 5V nominal.
Since the outputs are [open-collector][open-collector], I had to use a 1k-ohm pull-up resistor from the output to VCC.

The other issue is that I'm planning to drive a long strip of WS2812B NeoPixels, which draws far more power than the Raspberry Pi can supply.
I cut the end off an old 5V cell phone charger and put a header on it to simplify bread-boarding, then added a 3300uF 6.3V electrolytic capacitor that I had lying around.
The purpose of the capacitor is to stabilize the voltage being supplied to the strip when the current draw changes suddenly (e.g. when the lights are blinking on and off).

With the hardware interface figured out, I needed a way to generate the required Pulse-Width-Modulation (PWM) pattern to control the LED colors.
I did a lot of research about how this works and was considering doing it with a low-cost AVR microcontroller that would interface with the Raspberry Pi.
If you're interested in the details, you should check out [the NeoPixel posts on josh.com][josh_com].
This guy did some truly amazing work documenting how the WS2812B "NeoPixel" works, and how to interface efficiently with it.

In the end, it wasn't necessary, because [the rpi_ws281x project][rpi_ws281x] makes it possible to directly generate the required patterns using the Raspberry Pi's hardware PWM and Direct Memory Access (DMA) to.

[74ahct125]: https://www.adafruit.com/product/1787
[sn74als1035n]: http://www.mouser.com/ProductDetail/Texas-Instruments/SN74ALS1035N
[open-collector]: https://en.wikipedia.org/wiki/Open_collector
[josh_com]: http://wp.josh.com/category/neopixel
[rpi_ws281x]: https://github.com/jgarff/rpi_ws281x

## Building the Elixir Project and Interfacing with C code

This is the part of the project where I learned a lot about the basics of Elixir projects and a few more obscure details about building native C code as part of a project.

To integrate the `rpi_ws281x` C library with my Elixir code, I chose to write a little C wrapper around it so that it would accept binary pixel data on `STDIN`, taking come configuration parameters on the command-line.
From there, I used a `Port` in Elixir to 'safely' talk to this C code without risking a catastrophic failure of the whole Erlang VM by loading the C code directly using the `NIF` method.

Here's how that looks.
Also note the use of `:code.priv_dir/1`, which takes the name of the specified Release and returns the file system path to the `priv/` directory within your packaged Application.

``` elixir nerves_io_neopixel_driver.ex
defmodule Nerves.IO.Neopixel.Driver do

  use GenServer
  require Logger

  def start_link(settings, opts) do
    Logger.debug "#{__MODULE__} Starting"
    GenServer.start_link(__MODULE__, settings, opts)
  end

  def init(settings) do
    Logger.debug "#{__MODULE__} initializing: #{inspect settings}"
    pin   = settings[:pin]
    count = settings[:count]

    cmd = "#{:code.priv_dir(:nerves_io_neopixel)}/rpi_ws281x #{pin} #{count}"
    port = Port.open({:spawn, cmd}, [:binary])
    {:ok, port}
  end

  def handle_call({:render, pixel_data}, _from, port) do
    Logger.debug "#{__MODULE__} rendering: #{inspect pixel_data}"
    Port.command(port, pixel_data)
    {:reply, :ok, port}
  end

end
```

The other secret sauce was something I adapted from [Frank Hunleth's elixir_ale project][elixir_ale], which also uses some C code for low-level hardware interfacing.
In your `mix.exs`, you just have to define a special `Compile` task (mine is called `Ws281x`), then add it to the `compilers` list further down.

``` elixir mix.exs
defmodule Mix.Tasks.Compile.Ws281x do
  def run(_) do
    0 = Mix.Shell.IO.cmd("make priv/rpi_ws281x")
    Mix.Project.build_structure
    :ok
  end
end

defmodule Nerves.IO.Neopixel.Mixfile do
  use Mix.Project

  def project, do: [
    # ...
    compilers: [:Ws281x, :elixir, :app],
  ]
  # ...
end
```

This tells `mix` to shell-out to `make` to build `priv/rpi_ws281x`, which is the target of the C code I wrapped around the `rpi_ws281x` library.
The `priv` directory in your project folder ends up getting packaged into the Erlang Release that is burned into the SD card, and accessible using the `:code.priv_dir/1` function mentioned earlier.
Here is what I have in the `Makefile` to make that work:

``` makefile Makefile
include $(NERVES_ROOT)/scripts/nerves-elixir.mk

priv/rpi_ws281x: src/dma.c src/pwm.c src/ws2811.c src/main.c
  @mkdir -p priv
  $(CC) $(CFLAGS) -o $@ $^
```

[elixir_ale]: https://github.com/fhunleth/elixir_ale

## Controlling NeoPixels from Elixir

If you want to dive in and try running the code, download [the `nerves_io_neopixel` repository][nerves_io_neopixel]:

``` console bash
mkdir ~/projects
cd ~/projects
git clone git@github.com:GregMefford/nerves_io_neopixel.git
cd nerves_io_neopixel
```

Now, assuming that you checked out `nerves-system-br` to your home directory and did the `make` step earlier, you can `source` the environment script to set up the cross-compilers, then build the project.

``` console bash
source ~/nerves-system-br/nerves-env.sh
make
```

If you're doing this on a Linux VM with a Windows host, you also need to take one more step to generate the `.img` file that you need to burn to the SD card:

``` console bash
fwup -a -i _images/nerves_io_neopixel.fw -d _images/nerves_io_neopixel.img -t complete
```

This takes the efficiently-packed "firmware" file with a `.fw` extension and formats it into a much larger file with the appropriate blank-space offsets so that it can be booted on the target.

``` console bash
 18M nerves_io_neopixel.fw
329M nerves_io_neopixel.img
```

From there, you can copy the `.img` file to the Windows host using WinSCP, burn it to an SD card using Win32DiskImager, and boot from it on the Raspberry Pi.

{% img center /images/posts/2016-01-22-driving-neopixels-with-elixir-and-nerves/winscp.png Copying the img file with WinSCP %}

{% img center /images/posts/2016-01-22-driving-neopixels-with-elixir-and-nerves/win32diskimager.png Burning the SD card with Win32DiskImager %}

Once the Pi boots, it loads `iex` but doesn't do anything with the LEDs.
To make something display, you have to `setup` which I/O pin to use and how many LEDs are chained together, then `render` something to them:

``` elixir iex
Alias Nerves.IO.Neopixel
{:ok, pid} = Neopixel.setup pin: 18, count: 3
Neopixel.render(pid, <<255, 0, 0>> <> <<0, 255, 0>> <> <<0, 0, 255>>)
```

{% img center /images/posts/2016-01-22-driving-neopixels-with-elixir-and-nerves/nerves_neopixel_rgb.jpg Red, Green, and Blue NeoPixels %}

The second argument to the `render` function is a binary representing the RGB values of each LED.
I have just concatenated three 3-byte binaries here so it's easier to see where each LED's configuration begins and ends.
It's not pretty, and it would be tedious to do anything very complicated with just this interface, but it works!

To demonstrate something a bit more fun, I added a `scan` function that uses the `render` interface to draw a single red light sweeping back and forth across the strip, Battlestar Galactica style.
This initializes a strip with 72 LEDs (since I happened to have a half-meter strip of [the 144-LED-per-meter variety][144_meter_strip]) and scans a red dot across them 5 times, displaying each frame for 10 ms.

``` elixir iex
Alias Nerves.IO.Neopixel
{:ok, pid} = Neopixel.setup pin: 18, count: 72
Neopixel.scan(pid, 10, 72, 5)
```

{% img center /images/posts/2016-01-22-driving-neopixels-with-elixir-and-nerves/nerves_scanner.gif Red Light Scanning Across the Strip %}

[nerves_io_neopixel]: https://github.com/GregMefford/nerves_io_neopixel
[144_meter_strip]: https://www.adafruit.com/products/1506
