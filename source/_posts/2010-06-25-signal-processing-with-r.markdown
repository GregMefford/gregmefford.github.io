---
layout: post
title: "Signal Processing with R"
date: 2010-06-25 14:40:39 -0400
comments: true
categories:
- R
- Measurements
- Signal Processing
- Hardware
- Software
---

I have been meaning to check out [R] for a few years now, but I got busy and I just never got around to doing much more than installing the MacOS X version on my laptop.
I finally got my excuse to try it when I needed to start working on signal processing of the measurements I had been gathering.
My usual go-to tool for this type of task was [MATLAB].
However, on this fateful day, when I tried to launch my copy of MATLAB, I got some kind of message saying that I needed to renew my license.
At some point in the past, I had bought a one-year student edition of MATLAB, so this wasn't a huge surprise.
However, that message was all it took to get me to finally check out R.

[R]: http://www.r-project.org/
[MATLAB]: http://www.mathworks.com/

<!-- more -->

## What is R?

Despite its rather silly name, R is a phenominal piece of software for analyzing data and generating high-quality charts and graphs.
It has a great user community, extensions to support just about any kind of data analysis you can think of, and best of all, it's free (as in "free beer" as well as "free speech").
R offers a variety of functions for statistical analysis and signal processing of large data sets.

## Problem Description

The data set to be analyzed is a set of measurements from an oscilloscope.
Each measurement represents a trace of the same analog signal, but they are not time-aligned and each has a considerable amount of noise.
The goal of this experiment is to time-align these data samples and perform some statistical analysis on them to pull some kind of signal out of the noise.

## Loading the Data

The first task in any data-processing pipeline is the mundane step of importing the raw data from the data source into the data processing tool.
In this case, the data source is the Agilent oscilloscope being used to gather the measurements by converting the continuous-time analog signal of interest into a series of digital samples with 8 bits of resolution.
The wave forms are saved in "segmented time" mode so that multiple traces can be acquired in the oscilloscope before performing the alignment and statistical analysis.
These segmented time acquisitions are saved as a tab-delimited file with a tabular structure.
Each column of data represents a memory segment, which is a single trace.
Each trace consists of approximately 25,000 samples, stored as rows in the data file.

To load this type of data file into R and convert a trace into a time series for plotting and manipulation, the process is very straightforward:

``` r Reading a CSV file
traces <- read.table("16traces.txt")
```

It took me a while to figure out how to set up the margins on the plot, but it turns out to be simple once you find the proper command to set the graphics defaults.
`mar(bottom, left, top, right)` sets the margins around the plot, and `mgp(title, x, y)` sets the margin lines for the title and axes.

``` r Formatting and Plotting a Time Series
par(mar=c(3, 3, 3, 1), mgp=c(1, 0, 0), xaxt="n", yaxt="n")
plot(timeseries, main="Single Raw Trace", xlab="Time", ylab="Current")
```

Plotting to a PNG image is also possible, with the result following:

``` r Plotting to a PNG Image File
png("single_raw_trace.png", width=600, height=200)
  par(mar=c(3, 3, 3, 1), mgp=c(1, 0, 0), xaxt="n", yaxt="n")
  plot(timeseries, main="Single Raw Trace", xlab="Time", ylab="Current")
dev.off()
```

{% img center /images/posts/2010-06-25-signal-processing-with-r/single_raw_trace.png A single time-series measurement %}

## Simplifying the Data

Now that we have the basics laid out, onward to some real work.

The first step needed to process the measurements is to smooth out the data to allow the lower-frequency features to be more visible.
To do this, I am going to use R's `lowess` function.
The best way to describe what this does is with an example:

``` r Demonstrating Lowess Smoothing
png("lowess_smoothing.png", width=600, height=200)
  par(mar=c(3, 3, 3, 1), mgp=c(1, 0, 0), xaxt="n", yaxt="n")
  plot(timeseries, col="grey",
    main="Lowess Smoothing", xlab="Time", ylab="Current"
  )
  lines(lowess(timeseries, f=.07))
dev.off()
```

{% img center /images/posts/2010-06-25-signal-processing-with-r/lowess_smoothing.png Lowess Smoothing %}

## Alignment

The next step is pure magic.

What we need to do is align each measurement trace collected so that corresponding features can be compared and analyzed.
The plan is to use R's `ccf` function, which plots the correlation between two functions vs. the lag between them.

``` r Correlating Two Traces while Varying the Lag Between Signals
ts1.low = lowess(ts(traces[<sup class="footnote"><a href="#fn1">1</a></sup>]), f=.07)
ts5.low = lowess(ts(traces[<sup class="footnote"><a href="#fn5">5</a></sup>]), f=.07)
png("correlation_t1_t5.png", width=600, height=400)
  par(mar=c(4, 4, 4, 1), mgp=c(1, 1, 1))
  plot(
    ccf(ts1.low$y, ts5.low$y, lag.max=length(ts1)/2),
    main="Correlation: Trace 1 vs Trace 5", xlab="Lag", ylab="Correlation"
  )
dev.off()
```

{% img center /images/posts/2010-06-25-signal-processing-with-r/correlation_t1_t5.png Correlation: Trace 1 vs Trace 5 %}

This plot is neat to look at, but what we really want is to find the absolute maximum of this function, which represents the offset that will cause the two traces to be aligned as well as possible.

``` r Finding the Lag Value that Maximizes Correlation
  results = ccf(ts1.low$y, ts5.low$y, lag.max=length(ts1)/2)
  results$lag[which.max(results$acf)]
```

The preceding code returns `536` for the data shown here, which passes the "sniff-test" according to the plot of the `ccf` function shown in the previous section.

If we lag the second plot using this amount, they should line up:

``` r Plotting the Two Traces Using the Maximum-Correlation Lag Value
png("auto-aligned_traces.png", width=600, height=400)
  par(mar=c(3, 3, 3, 1), mgp=c(1, 0, 0), xaxt="n", yaxt="n")
  best_lag = results$lag[which.max(results$acf)]
  ts1.lagged.low = lowess(lag(ts1, best_lag), f=.07)
  plot(ts1.lagged.low, col="grey",
    main="Auto-Aligned Traces", xlab="Time", ylab="Current"
  )
  lines(ts5.low)
dev.off()
```

{% img center /images/posts/2010-06-25-signal-processing-with-r/auto-aligned_traces.png Auto-Aligned Traces %}

This code starts to get pretty ugly, but the resulting figure shows just how nicely-aligned these features are, using a more fine-grained smoothing function to show a bit more detail:

``` r Plotting the Aligned Traces with Less Smoothing
png("hires_aligned_traces.png", width=600, height=200)
  par(mar=c(3, 3, 3, 1), mgp=c(1, 0, 0), xaxt="n", yaxt="n")
  ts1.low = lowess(ts1, delta=10, f=.01)
  ts5.low = lowess(ts5, delta=10, f=.01)
  ts1.aligned = ts(ts1.low$y[best_lag:length(ts1.lagged.low$y)])
  ts5.aligned = ts(ts5.low$y)
  plot(ts1.aligned, col="grey",
    main="High-Resolution Auto-Aligned Traces", xlab="Time", ylab="Current"
  )
  lines(ts5.aligned)
dev.off()
```

{% img center /images/posts/2010-06-25-signal-processing-with-r/hires_aligned_traces.png High-Resolution Auto-Aligned Traces %}

That's all for now!
Coming up next time: cropping out just the interesting part of all the traces to build up a useful data set.
