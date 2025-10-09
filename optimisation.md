---
title: Optimising workflow performance
teaching: 20
exercises: 20
---

::::::::::::::::::::::::::::::::::::::: objectives

- Understand CPU, RAM and I/O bottlenecks
- Understand the `threads` declaration
- Use common Unix tools to look at resource usage

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::: questions

- What compute resources are available on my system?
- How do I define jobs with more than one thread?
- How do I measure the compute resources being used by a workflow?
- How do I run my workflow steps in parallel?

::::::::::::::::::::::::::::::::::::::::::::::::::

## Processes, threads and processors

Some definitions:

- **Process**:
  A running program
  (in our case, each Snakemake job can be considered one process)
- **Threads**:
  Each process has one or more threads which run in parallel
- **Processor**:
  Your computer has multiple *CPU cores* or processors,
  each of which can run one thread at a time

These definitions are a little simplified,
but fine for our needs.
The operating system kernel shares out threads among processors:

- Having *fewer threads* than *processors* means you are not fully using all your CPU cores
- Having *more threads* than *processors* means threads have to "timeslice" on a core
  which is generally suboptimal

If you tell Snakemake how many threads each rule will use,
and how many cores you have available,
it will start jobs in parallel to use all your cores.
In the diagram below,
five jobs are ready to run and there are four system cores.

![][fig-threads]

## Listing the resources of your machine

So,
to know how many threads to make available to Snakemake,
we need to know how many CPU cores we have on our machine.
On Linux,
we can find out how many cores you have on a machine
with the `lscpu` command.

```shellsession
$ lscpu
Architecture:             aarch64
  CPU op-mode(s):         32-bit, 64-bit
  Byte Order:             Little Endian
CPU(s):                   4
  On-line CPU(s) list:    0-3
Vendor ID:                ARM
  Model name:             Cortex-A72
    Model:                3
    Thread(s) per core:   1
```

There we can see that we have four CPU cores,
each of which can run a single thread.

On macOS meanwhile,
we use the command `sysctl -n hw.ncpu`:

```shellsession
$ sysctl hw.ncpu
hw.ncpu: 8
```

In this case,
we see that this Mac has eight cores.

Likewise find out the amount of RAM available:

```bash
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           3.7Gi       1.1Gi       110Mi        97Mi       2.6Gi       2.6Gi
Swap:          199Mi       199Mi        60Ki
```

In this case,
the machine has 3.7GiB of total RAM.

On macOS,
the command is `sysctl -h hw.memsize`:

```shellsession
$ sysctl -h hw.memsize
hw.memsize: 34,359,738,368
```

This machine has around 34GB of RAM in total.
Dividing by the number of bytes in 1GiB
($1024^3$ bytes),
that becomes 32GiB RAM.

We don't want to use all of this RAM,
but if we don't mind other applications being unresponsive while our workflow runs,
we can use the majority of it.

Finally,
to check the available disk space,
on the current partition:

```bash
$ df -h .
```

(or `df -h` without the `.` to show all partitions)
This is the same on both macOS and Linux.

## Parallel jobs in Snakemake

You may want to see the relevant part of [the Snakemake documentation
](https://snakemake.readthedocs.io/en/stable/snakefiles/rules.html#threads).

We'll force all the intermediary steps to re-run
by using the `--forceall` flag to Snakemake
and time the whole run using the `time` command.

```bash
$ time snakemake --cores 1 --forceall -- assets/plots/spectrum.pdf
real	3m10.713s
user	1m30.181s
sys	0m8.156s
```

:::::::::::::::::::::::::::::::::::::::  challenge

## Measuring how concurrency affects execution time

What is the *wallclock time* reported by the above command?
We'll work out the average for everyone present,
or if you are working through the material on your own,
repeat the measurement three times to get your own average.

Now change the Snakemake concurrency option to  `--cores 2` and then `--cores 4`.
Finally,
try using every available core on your machine,
using `--cores all`.

- How does the total execution time change?
- What factors do you think limit
  the power of this setting to reduce the execution time?

:::::::::::::::  solution

## Solution

The time will vary depending on the system configuration
but somewhere around 150&ndash;200 seconds is expected,
and this should reduce to around 75&ndash;100 secs with `--cores 2`
but depending on your computer,
higher `--cores` might produce diminishing returns.

Things that may limit the effectiveness of parallel execution include:

- The number of processors in the machine
- The number of jobs in the DAG which are independent
  and can therefore be run in parallel
- The existence of single long-running jobs
- The amount of RAM in the machine
- The speed at which data can be read from and written to disk

:::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::

There are **a few gotchas** to bear in mind when using parallel execution:

1. Parallel jobs will use more RAM.
   If you run out then either your OS will swap data to disk,
   or a process will crash.
2. Parallel jobs may trip over each other
   if they try to write to the same filename at the same time
   (this can happen with temporary files).
3. The on-screen output from parallel jobs will be jumbled,
   so save any output to log files instead.

## Multi-thread rules in Snakemake

In the diagram at the top,
we showed jobs with 2 and 8 threads.
These are defined by adding a `threads:` block to the rule definition.
We could do this for the `ps_mass` rule:

```source
# Compute pseudoscalar mass and amplitude, read plateau from metadata,
# and plot effective mass
rule ps_mass:
    input: "raw_data/beta{beta}/out_corr"
    output:
        data="intermediary_data/beta{beta}/corr.ps_mass.json.gz",
        plot=multiext(
            "intermediary_data/beta{beta}/corr.ps_eff_mass",
            config["plot_filetype"],
        ),
    log:
        messages="intermediary_data/beta{beta}/corr.ps_mass.log",
    params:
        plateau_start=lookup(within=metadata, query="beta == {beta}", cols="ps_plateau_start"),
        plateau_end=lookup(within=metadata, query="beta == {beta}", cols="ps_plateau_end"),
    conda: "envs/analysis.yml"
    threads: 4
    shell:
        "python -m su2pg_analysis.meson_mass {input} --output_file {output.data} --plateau_start {params.plateau_start} --plateau_end {params.plateau_end} --plot_file {output.plot} --plot_styles {config[plot_styles]} |& tee {log.messages}"
```

You should explicitly use `threads: 4` rather than `params: threads = "4"` 
because Snakemake considers the number of threads when scheduling jobs.
Also,
if the number of threads requested for a rule
is less than the number of available processors
then Snakemake will use the lower number.

Snakemake uses the `threads` variable
to set common environment variables like `OMP_NUM_THREADS`.
If you need to pass the number explicitly to your program,
you can use the `{threads}` placeholder to get it.

:::::::::::::::::::::::::::::::::::::::::  callout

## Fine-grained profiling

Rather than timing the entire workflow,
we can ask Snakemake to benchmark an individual rule.

For example,
to benchmark the `ps_mass` step we could add this to the rule definition:

```source
rule ps_mass:
    benchmark:
        "benchmarks/ps_mass.beta{beta}.txt"
    ...
```

The dataset here is so small that the numbers are tiny,
but for real data this can be very useful as it shows
time, memory usage and IO load for all jobs.

::::::::::::::::::::::::::::::::::::::::::::::::::

## Running jobs on a cluster

Learning about clusters is beyond the scope of this course,
but can be essential for more complex workflows working with large amounts of data.

When working with Snakemake,
there are two options to getting the workflow running on a cluster:

1. Similarly to most tools,
   we may install Snakemake on the cluster,
   write a job script,
   and execute Snakemake on our workflow inside a job.

2. We can teach Snakemake how to run jobs on the cluster,
   and run our workflow from our own computer,
   having Snakemake do the work of submitting and monitoring the jobs for us.

To run Snakemake in the second way,
someone will need to determine the right parameters for your particular cluster
and save them as a profile.
Once this is working,
you can share the profile with other users on the cluster,
so discuss this with your cluster sysadmin.

Instructions for configuring the Slurm executor plugin can be found in
the [Snakemake plugin catalog](https://snakemake.github.io/snakemake-plugin-catalog/),
along with the `drmaa`,
`cluster-generic` and `cluster-sync` plugins
which can support PBS, SGE and other cluster schedulers.

![][fig-cluster]

::::::::::::::::::::::::::::::: instructor

## Running on cluster and cloud

Running workflows on HPC or Cloud systems could be a whole course in itself.
The topic is too important not to be mentioned here,
but also complex to teach because you need a cluster to work on.

If you are teaching this lesson and have institutional HPC
then ideally you should liaise with the administrators of the system
to make a suitable installation of a recent Snakemake version
and a profile to run jobs on the cluster job scheduler.
In practise this may be easier said than done!

If you are able to demonstrate Snakemake running on cloud as part of a workshop
then we'd much appreciate any feedback on how you did this and how it went.

::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::  callout

## Cluster demo

A this point in the course there may be a cluster demo...

::::::::::::::::::::::::::::::::::::::::::::::::::

[comment]: # (
Photo credit: Cskiran
Sourced from Wikimedia Commons
CC-BY-SA-4.0
)



[fig-threads]: fig/snake_threads.svg {alt='Representation of a computer with four microchip icons indicating four available cores.
To the right are five small green boxes representing Snakemake jobs
and labelled as wanting 1, 1, 1, 2 and 8 threads respectively.
'}

[fig-cluster]: fig/cluster.jpg {alt='A photo of some high performance computer hardware
racked in five cabinets in a server room.
Each cabinet is about 2.2 metres high and 0.8m wide.
The doors of the cabinets are open to show the systems inside.
Orange and yellow cabling is prominent,
connecting ports within the second and third racks.
'}


:::::::::::::::::::::::::::::::::::::::: keypoints

- To make your workflow run as fast as possible,
  try to match the number of threads to the number of cores you have
- You also need to consider RAM, disk, and network bottlenecks
- Profile your jobs to see what is taking most resources
- Use `--cores all` to enable using all CPU cores
- Snakemake is great for running workflows on compute clusters

::::::::::::::::::::::::::::::::::::::::::::::::::
