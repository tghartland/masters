## Week 13
###### February 19th-25th 2018

### Continuing

I've done a lot of work and am now writing up all at once. Hopefully I include everything
and get things in about the right oder.

I generated a bunch more data before last week's meeting and made a new brazil plot,
only it was much worse than the one I had previously, despite having more data:

![image](https://github.com/H4rtland/masters/blob/master/week13/imgs/brazil-51320.png "")

Because I was using the histogram-calculated RMS of the distribution as the 1 sigma value,
in cases where the distribution was close to 0 on the left with a long tail off to the right
it was causing the mean-2\*rms to be < 0. This is why the yellow band is getting pulled
down to the lower axis on the plot.

The fix for this is to work out the lower 1 and 2 sigma limits and the higher 1 and 2 sigma limits
separately, via converting to another cumulative distribution.

![image](https://github.com/H4rtland/masters/blob/master/week13/imgs/stat1.png "")

![image](https://github.com/H4rtland/masters/blob/master/week13/imgs/stat2.png "")

Source: http://pdg.lbl.gov/2017/reviews/rpp2017-rev-statistics.pdf

The 1sigma and 2sigma alpha values are what we will use, divided by 2 to account for
it being a two-tailed test.

The big concern now is that the resolution of the histograms for the higher mass tests
will not be enough to get accurate values for these errors. To try and improve this I have
changed the iteration step size in the likelihood testing to be as follows

* default --> 5 events
* q\* mass > 4000 GeV --> 1 event
* q\* mass > 5000 GeV --> 0.05 events

These values are mostly just guesses by me, but they cover two orders of magnitude difference,
which is about the same as the difference between the 2000 GeV 95% limit and the
7000 GeV 95% limit, so it might be reasonable.

The brazil plot temporary histograms are also binned with the above widths. THe histograms
are still used for binning the data, and to calculate the mean. I am choosing to keep using
this mean rather than finding a median from the 0.5 height on the cumulative distribution.
I think that the mean is probably more accurate because the cumulative distribution might have
two points that just step over the 0.5 value, meaning that either one is not a great
representation of the median.

### The blue line

Rather than having an equation for the blue line, as I thought last week, it's actually
just something derived from the individual simulated data files.

The brazil plot code opens each q\* mass file, gets the right histogram, and calculates
the integral of it. That number, divided by 30 (the 30 fb histogram), divided by 1000
(to convert to pb) is the value for the blue line at that mass.

### Optimisations

I ran some jobs this week after I changed the step size for the likelihood testing.
The high q\* mass files which were running with a step of 0.05 ran for a long time.
Like 5 hours. The reason was that I wasn't allowing it to exit out of the loop
until the N value was at least some value (which was 500 at the time). Since all the values
of the 95% limit were more like 20, it was wasting a bunch of time testing these higher values.
So I've removed the N value check completely, and now it only checks to see if the maximum
value in the cumulative distribution is changing by less than 0.1%. For limit values of ~20,
it now only has to check up to about ~30.

Another reason that the jobs took so long was that they were running for 1500 simulations.
Normally I don't watch the jobs as they're running, but of course when it was 3 hours
in and they still weren't done I was paying more attention to them than usual.
It was at this time that I realised that `condor_q` has a column for memory usage,
and that each of my jobs was using around 800Mb each. This really doesn't make sense to me.
Each job is using a pythia file of about 6 kilobytes, and a q\* file of about 3 megabytes,
so the raw data isn't enough to contribute that. I wouldn't be surprised if someone told me
that it was just ROOT's fault, and that it somehow always manages to use that much memory.

I thought that it might just be some sort of conflict between python memory management
and c++ management which could be leaking memory in the main loop. It would make sense,
since the current code is basically the previous code but with a loop that says "do this
1000 times", so I started by looking there. First I made sure that files were being closed
after they had been read from. Then I moved the input file reading to outside the loop entirely.
Still 800mb memory usage.

Oh, and to test for some kind of baseline I submitted a job with a new script which did nothing
but wait for 30 minutes and then exit, and that used ~100mb of memory. That surprised me.

Running on lapa, I found that the cause for the empty job seemed to be "locale-archive".

```
(root-py2)[thartland@lapa week13]$ ./job3.sh &
[1] 3681454
(root-py2)[thartland@lapa week13]$ pmap 3681454
3681454:   /bin/bash -f ./job3.sh
0000000000400000    852K r-x--  /bin/bash
00000000006d4000     36K rw---  /bin/bash
00000000006dd000     24K rw---    [ anon ]
00000000008dc000     36K rw---  /bin/bash
0000000000956000    132K rw---    [ anon ]
000000394fe00000    128K r-x--  /lib64/ld-2.12.so
0000003950020000      4K r----  /lib64/ld-2.12.so
0000003950021000      4K rw---  /lib64/ld-2.12.so
0000003950022000      4K rw---    [ anon ]
0000003950200000   1576K r-x--  /lib64/libc-2.12.so
000000395038a000   2048K -----  /lib64/libc-2.12.so
000000395058a000     16K r----  /lib64/libc-2.12.so
000000395058e000      8K rw---  /lib64/libc-2.12.so
0000003950590000     16K rw---    [ anon ]
0000003950e00000      8K r-x--  /lib64/libdl-2.12.so
0000003950e02000   2048K -----  /lib64/libdl-2.12.so
0000003951002000      4K r----  /lib64/libdl-2.12.so
0000003951003000      4K rw---  /lib64/libdl-2.12.so
0000003961e00000    116K r-x--  /lib64/libtinfo.so.5.7
0000003961e1d000   2044K -----  /lib64/libtinfo.so.5.7
000000396201c000     16K rw---  /lib64/libtinfo.so.5.7
0000003962020000      4K rw---    [ anon ]
00007fd1559b7000  96852K r----  /usr/lib/locale/locale-archive
00007fd15b84c000     12K rw---    [ anon ]
00007fd15b87b000     28K r--s-  /usr/lib64/gconv/gconv-modules.cache
00007fd15b882000      4K rw---    [ anon ]
00007ffd675e1000     84K rw---    [ stack ]
00007ffd675f6000      4K r-x--    [ anon ]
ffffffffff600000      4K r-x--    [ anon ]
 total           106116K
```

At the very least that's read-only so it should probably be shared memory.

Now checking the actual python code

```
(root-py2)[thartland@lapa week13]$ python limit_dist.py abc123 abc123.8 3000 > outfile2.txt 2>&1 &
[1] 3686030
(root-py2)[thartland@lapa week13]$ pmap -x 3686030 | sort -k 2 -n -r
00007f26b4283000   96852      40       0 r----  locale-archive
00007f26a0021000   65404       0       0 -----    [ anon ]
00000000019e6000   43988   41648   41648 rw---    [ anon ]
00007f26a5ca9000   10240      28      28 rw---    [ anon ]
00007f26a7359000    8212       8       0 r--s-  passwd
00007f26a4523000    7548    2924       0 r-x--  libGui.so.5.34
0000003955a00000    7332    4180       0 r-x--  libCore.so.5.34
00007f269f5d4000    6260       8       0 r--s-  group
000000395a200000    5360    2028       0 r-x--  libHist.so.5.34
0000003953e49000    5340    3108    3108 rw---    [ anon ]
00007f26b073a000    5188     116       0 r-x--  liblapack.so.3.0
00007f26b2e91000    5136     312       0 r-x--  libatlas.so.3.0
00007f26a5892000    4184       8       8 rw---    [ anon ]
0000003957e00000    3076    1340       0 r-x--  libRIO.so.5.34
0000003959200000    2448    1092       0 r-x--  libMathCore.so.5.34
0000003959800000    2364     736       0 r-x--  libMatrix.so.5.34
0000003953a00000    2308    1440       0 r-x--  libCint.so.5.34
0000003958400000    2248     920       0 r-x--  libTree.so.5.34
00007f26a6d01000    2140     628       0 r-x--  vector.so.5.34
00007f26b3b4f000    2048       0       0 -----  multiarray.so
00007f26b35be000    2048       0       0 -----  libptcblas.so.3.0
00007f26b3395000    2048       0       0 -----  libatlas.so.3.0
00007f26b2c8d000    2048       0       0 -----  datetime.so
00007f26b2a75000    2048       0       0 -----  umath.so
00007f26b25af000    2048       0       0 -----  libffi.so.5.0.6
00007f26b23a6000    2048       0       0 -----  \_struct.so
00007f26b1f94000    2048       0       0 -----  itertoolsmodule.so
00007f26b1d89000    2048       0       0 -----  cPickle.so
..............................
...........many.more..........
..............................
----------------  ------  ------  ------
total kB          630176   81008   50840
```

Again locale-archive is at the top. [anon] is memory allocated by the program, and then following
that it's just an almost neverending list of libs and python modules.

The conclusion is: I don't know where all this memory is going, and I don't have any other choice
but to keep submitting 50 jobs at a time using almost a gigabyte of memory each.

For comparison, I've asked Simon to run one job of his code to see if the pure c++ is
any better. While checking that we discovered someone else's single job running using
73Gb of memory all by itself, so I don't feel so bad any more.

And, now that condor has refreshed the memory usage, Simon's job is using ~730mb of memory.
So it's just ROOT.

### Back to it

Now that I have a higher resolution on the limit likelihood testing, and code to use this
to get 1sigma and 2sigma upper and lower errors in the brazil plotting code, I can
run on a new set of data and see what it looks like. All plots are included
in directory plt51329 above. The brazil plot is also included here.

![image](https://github.com/H4rtland/masters/blob/master/week13/imgs/brazil-51329.png "")

Now I'm interested to find out how close the y values in the cumulative sum are to the
actual values for one sigma, two sigma. I added some print statements to print out
x values, y value pairs being iterated over ifthat pair was being used for one of
the error values, and the particular sigma value at the end of the line to compare to.

For the 7000 GeV distribution, the relevant lines are:

```
12.25 0.03      0.02275
14.75 0.1872    0.15865
23.75 0.8596    0.84135
31.75 0.9776    0.97725
```

In particular that low 1sigma value is a couple of percent off.

Oh, I'm binning in 0.5 steps instead of 0.05. I'll change that and run again.

```
11.85 0.024 0.02275
14.25 0.1596 0.15865
23.15 0.8436 0.84135
31.55 0.9776 0.97725
``` 

Alright, that looks a lot better. I'll now check some of the other mass distributions.

3000 looks good.  
4000 looks alright.  
4500 has one value that looks a bit off `83.5 0.1632 0.15865`.  
5000 is the same `42.25 0.1636 0.15865`.  
5500 has the worst so far `28.75 0.1764 0.15865`.  
6000 is fine.  
6500 is a bit off `16.85 0.1664 0.15865`.

It's possible that I need to change the bin width values so that they get smaller earlier.
It's also strange that all the problem values I found were the lower 1sigma band, but I suppose
it's at that point that the size of each bin is the largest, so the cumulative distribution
has the largest steps at this point.

### The black line

The solid black line on the ATLAS plot represents the 95% likelihood cross section limit
at each mass point. To duplicate this myself I copied limit_dist.py as a new file limit_dist_data.py.
I changed things around in this file to use the actual ATLAS data file as the distribution
background rather than a distribution randomly generated from the Pythia data. Then the
likelihood calculation is ran at each mass point for 0 injected events, and the results
are written to a single file. This file is then read and the data points are added to the
brazil plot by plot_brazil.py.

![image](https://github.com/H4rtland/masters/blob/master/week13/imgs/brazil-51329-2.png "")

The data line on my plot follows exactly the path of the data line on the ATLAS plot.
Now the only difference is that the cross section limit of my simulated data is slightly too low,
because I am not yet including any of the additional uncertainties. But would this also change
the data line?
