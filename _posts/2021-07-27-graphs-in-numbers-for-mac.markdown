---
layout: post
title: "Graphs in Numbers for Mac"
date: 2021-07-27
---
## Introduction
This is to note down some of the things I found unintuitive about graphing using Numbers. I'm new to Numbers and some of these will seem laughably obvious to some.


## My Problem
I was trying to graph a dollar balance over time. Here is some example data:
![Example data](/assets/data.png){: width="250"}

Notice the dates of the samples are not regular. There are gaps in time because I didn't bother to record the balance on all dates.  I don't want to graph this data evenly spread out over the x-axis like I was graphing count of a category - I want the data points spead out according to the date of each point.

My first attempt was to select the data and insert a Line graph:

![Example data](/assets/line graph.png){: width="250"}

Numbers treats the dates a categories - if column A was "Cat", "Dog", "Fish", etc, then the result would be the same.  This is different to Excel which seems to be better at recognising the dates are not categories.

So, the first thing to realize is that most graph types are what could be called "category" graphs - they show counts of items in a category.  These are the common types:
- Column / Bar (plus the stacked varieties)
- Line / Area
- Pie / Doughnut

Scatter and Bubble graph types are different - they graph one series against the other.  Only Scatter and Bubble will treat the dates as dates.  ie. space the dates across the x-axis according to any gaps.  The other types interpret the dates as category lables (just like "Oranges", "Apples", etc) and data points are spaced evenly over the x-axis.

One other wrinkle is that column A above is a "Header" column (also row 1 is a Header row).  These are treated specially in Numbers which creates a named range based on the header name.  Headers are not supposed to be used in calculations (though they seem to be if referenced in formulars). The text in headers becomes the category names in graphs.  For the x-y graphs (scatter and bubble), though, the headers are ignored.  You must select data references that are not header rows/columns for these graph types.  The header data is ignored.

To acheive my aim I used this data (note - no headers) and a scatter plot type:

![scatter plot](/assets/scatter plot.png){: width="500"}

And there you have it.