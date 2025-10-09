---
title: Awkward corners
teaching: 20
exercises: 20
---

::::::::::::::::::::::::::::::::::::::: objectives

- Understand how Snakemake being built on Python allows us 
  to work around some shortcomings of Snakemake in some use cases.
- Understand how to handle trickier metadata and input file lookups.
- Be able to avoid Snakemake re-running a rule when this is not wanted.

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::: questions

- How can I look up metadata based on wildcard values?
- How can I select different numbers of input files
  depending on wildcard values?
- How can I tell Snakemake not to regenerate a file?

::::::::::::::::::::::::::::::::::::::::::::::::::

## Beyond the pseudoscalar: Input functions

Recall the rule that we have been working on
to fit the correlation function of the pseudoscalar meson:

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
    shell:
        "python -m su2pg_analysis.meson_mass {input} --output_file {output.data} --plateau_start {params.plateau_start} --plateau_end {params.plateau_end} --plot_file {output.plot} --plot_styles {config[plot_styles]} |& tee {log.messages}"
```

Now,
we are frequently interested in more symmetry channels than just the pseudoscalar.
In principle,
we could make a copy of this rule,
and change `ps` to,
for example
`v` or `av`.
(We did this in an earlier challenge for the vector channel.)
However,
just like adding more ensembles,
this rapidly makes our Snakefiles unwieldy and difficult to maintain.
How can we adjust this rule so that it works for any channel?

The first step is to replace `ps` with a wildcard.
The `output` should then use `corr.{channel}_mass.json`
rather than `corr.ps_mass.json`,
and similarly for the log.
We will also need to pass an argument
`--channel {wildcards.channel}`
in the `shell` block,
so that the code knows what channel it is working with.
Since the rule is no longer only for the `ps` channel,
it should be renamed,
for example to `meson_mass`.

What about the plateau start and end positions?
Currently these are found using a call to `lookup`,
with the column hardcoded to `"ps_plateau_start"` or `"ps_plateau_start"`.
We can't substitute a wildcard in here,
so instead we need to make use of an _input function_.

When Snakemake is given a Python function as a `param`, `input`, or `output`,
then it will call this function,
passing in the parsed wildcards,
and use the result of the function call as the value.
For example,
I could define a function:

```python
def get_mass(wildcards):
    return 1.0
```

and then make use of this within a rule as:

```source
rule test_mass:
    params:
        mass=get_mass,
    ...
```

This would then set the parameter `params.mass` to `1.0`.

How can we make use of this for getting the plateau metadata?

```python
def plateau_param(position):
    """
    Return a function that can be used to get a plateau position from the metadata.
    `position` should be `start` or `end`.
    """
    def get_plateau(wildcards):
        return lookup(
            within=metadata,
            query="beta == {beta}",
            cols=f"{wildcards['channel']}_plateau_{position}",
        )

    return get_plateau
```

Working inside out,
you will recognise the `lookup` call from before,
but the `cols` argument is now changed:
we make use of Python's [f-strings][f-strings]
to insert both the channel and the position (`start` or `end`).
We wrap this into a function called `get_plateau`,
which only takes the `wildcards` as an argument.
The `position` is defined in the outer function,
which creates and returns the correct version of `get_plateau`
depending on which `position` is specified.
We need to do this because
Snakemake doesn't give a direct way
to specify additional arguments to the input function.

To use this function within the rule,
we can use `plateau_start=plateau_param("start")` in the `params` block
for the plateau start position,
and similarly for the end.

::::::::::::::::::::::::::::::::::: instructor

## Closure alternatives

Here we choose to use a _closure_
(a function returned by another function,
where the former's behaviour depends on the arguments to the latter),
so that the same code can be used for both the start and the end of the plateau.
There are other ways to phrase this:
you could define a free function `get_plateau(wildcards, position)`,
and then in the rule definition,
use `functools.partial(get_plateau, position=...)` to set the `position`,
or use a lambda
`lambda wildcards: get_plateau(wildcards, ...)`.
We choose the closure here
because defining functions should already be familiar to most learners,
and passing functions as values needs to be learned anyway
(since we have to pass one to Snakemake),
whereas lambdas and `functools` may not be familiar,
and aren't needed elsewhere in the lesson.

:::::::::::::::::::::::::::::::::::::::

With these changes,
the full rule now becomes:

```
# Compute meson mass and amplitude, read plateau from metadata,
# and plot effective mass
rule meson_mass:
    input: "raw_data/beta{beta}/out_corr"
    output:
        data="intermediary_data/beta{beta}/corr.{channel}_mass.json.gz",
        plot=multiext(
            "intermediary_data/beta{beta}/corr.{channel}_eff_mass",
            config["plot_filetype"],
        ),
    log:
        messages="intermediary_data/beta{beta}/corr.{channel}_mass.log",
    params:
        plateau_start=plateau_param("start"),
        plateau_end=plateau_param("end"),
    conda: "envs/analysis.yml"
    shell:
        "python -m su2pg_analysis.meson_mass {input} --channel {wildcards.channel} --output_file {output.data} --plateau_start {params.plateau_start} --plateau_end {params.plateau_end} --plot_file {output.plot} --plot_styles {config[plot_styles]} |& tee {log.messages}"
```

We can test this for the vector mass:

```shellsession
snakemake --use-conda --cores all --printshellcmds intermediary_data/beta2.0/corr.v_mass.json.gz
```

:::::::::::::::::::::::::::::::::::::::::  instructor

## Change of command line

If learners skipped over the previous section,
note that we've changed the standard `snakemake` call we were previously using:
now,
we don't use `--forceall`,
so we only regenerate when necessary
(which makes the run quicker).
`--cores all`,
meanwhile,
tells Snakemake to use all available CPU cores.
In this case it doesn't make a difference,
since only one job is needed by this run,
but it's a useful default invocation for production use.

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::  challenge

## Restricted spectrum

Define a rule `restricted_spectrum`
that generates a plot of the pseudoscalar decay constant against its mass
(both in lattice units),
that only plots data where $\beta$ is below a specified value `beta0`,
which is included in the filename.

Use this to test plotting the spectrum for $\beta < 2.0$.

Hint:
Similarly to the above,
you may want to define a function that defines a function,
where the former takes the relevant fixed part of the filename
(`corr.ps_mass` or `pg.corr.ps_decay_const`)
as input,
and the latter the wildcards.

:::::::::::::::  solution

## Solution

We can constrain the data plotted by constraining the input data.
Similarly to for the example above,
we can use an input function for this,
but this time for the `input` data rather than to define `params`.

```
def spectrum_param(slug):
    def spectrum_inputs(wildcards):
        return [
            f"intermediary_data/beta{beta}/{slug}.json.gz"
            for beta in metadata[metadata["beta"] < float(wildcards["beta0"])]["beta"]
        ]

    return spectrum_inputs


rule restricted_spectrum:
    input:
        script="src/plot_spectrum.py",
        ps_mass=spectrum_param("corr.ps_mass"),
        ps_decay_const=spectrum_param("pg.corr.ps_decay_const"),
    output:
        plot=multiext("assets/plots/spectrum_beta{beta0}", config["plot_filetype"]),
    conda: "envs/analysis.yml"
    shell:
        "python {input.script} {input.ps_mass} {input.ps_decay_const} --y_observable f_ps --zero_y_axis --zero_x_axis --output_file {output.plot} --plot_styles {config[plot_styles]}"
```

To generate the requested plot would then be:

```shellsession
snakemake --use-conda --cores all --printshellcmds assets/plots/spectrum_beta2.0.pdf
```

:::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::

## Globbing

So far,
our workflow has been entirely deterministic in what inputs to use.
We have referred to
parts of filenames that are substituted in based on the requested outputs
as "wildcards".
You might be familiar with another meaning of the word "wildcard",
however:
the `*` and `?` characters that you can use in the shell
to match _any number of_ and _one or more_ characters respectively.

In general we would prefer to avoid using these in our workflows,
since we would like them to be reproducible.
If a particular input file is missing,
we would like to be told about it,
rather than the workflow silently producing different results.
(This also makes the failure cases easier to debug,
since otherwise we can hit cases where
we call our tools with an unexpected number of inputs,
and they fail with strange errors.)

However,
occasionally we do find ourselves needing to perform shell-style wildcard matches,
also known as _globbing_.
For example,
let's say that we would like to be able to plot
only the data that we have already computed,
without regenerating anything.
We can perform a wildcard _glob_ using Python's `glob` function.

At the top of the file,
add:

```
from glob import glob
```

Then we can add a rule:

```
rule quick_spectrum:
    input:
        script="src/plot_spectrum.py",
        ps_mass=glob(
            "intermediary_data/beta*/corr.ps_mass.json.gz",
        ),
        ps_decay_const=expand(
            "intermediary_data/beta*/pg.corr.ps_decay_const.json.gz",
        ),
    output:
        plot=multiext("intermediary_data/check_spectrum", config["plot_filetype"]),
    conda: "envs/analysis.yml"
    shell:
        "python {input.script} {input.ps_mass} {input.ps_decay_const} --y_observable f_ps --zero_y_axis --zero_x_axis --output_file {output.plot} --plot_styles {config[plot_styles]}"
```

Let's test this now

```shellsession
snakemake --use-conda --cores all --printshellcmds intermediary_data/check_spectrum.pdf
```

If you have recently purged the workflow output,
then this might raise an error as there are no data to plot.
Otherwise,
opening this resulting file in your PDF viewer,
you'll most likely see a plot with a reduced number of points compared to previous plots.
If you're lucky,
then all the points will be present.

:::::::::::::::::::::::::::::::::::::::::  callout

## Use with care in production!

This rule will only include
data already present at the start of the workflow run
in your plots.
This means if you start from clean,
as people looking to reproduce your work will,
then it will fail,
and even if you start from a partial run,
then some points will
(non-deterministically)
be omitted from your plots.

For final plots based on intermediary data,
always specify the `input` explicitly.

::::::::::::::::::::::::::::::::::::::::::::::::::

## Marking files as up to date

One of the useful aspects of Snakemake's DAG is that
it will automatically work out which files need updating
based on changes to their inputs,
and re-run any rules needed to update them.
Occasionally,
however,
this isn't what we want to do.

Consider the following example:

![][fig-asymmetric]

We have two steps to this workflow:
taking the input data and from it generating some intermediary data,
requiring 24 hours.
Then we take the resulting data and plot results from it,
taking 15 seconds.
If we make a trivial change to the input file,
such as reformatting white space,
we'd like to be able to test the output stage
without needing to wait 24 hours for the data to be regenerated.

Further,
if we share our workflow with others,
we'd like them to be able to reproduce the easy final stages
without being required to run the more expensive early ones.
This especially applies where the input data are very large,
and the intermediary data smaller;
this reduces the bandwidth and disk space required for those reproducing the work.

We can achieve this by telling Snakemake to "touch" the relevant file.
Similar to the `touch` shell command,
which updates the modification time of a file without making changes,
Snakemake's `--touch` option
updates Snakemake's records of when a file was last updated,
without running any rules to update it.

For example,
in the above workflow,
we may run

```shellsession
snakemake --touch data_file
snakemake --cores all --use-conda --printshellcmds plot_file
```

This will run only the second rule,
not the first.
Thiis will be the same regardless of whether `input_file` is even present,
or when it was last updated.


[fig-asymmetric]: fig/asymmetric-workflow.svg {alt='Diagram showing three boxes in a row,
  connected from left to right by arrows.
  The boxes are labeled
  "`input_file`",
  "`data_file`",
  and "`output_file`".
  The arrows are labelled
  "`generate`" (24 hours),
  and "`plot`" (15 seconds).
'}

[f-strings]: https://docs.python.org/3/reference/lexical_analysis.html#f-strings


:::::::::::::::::::::::::::::::::::::::: keypoints

- Use Python input functions that take a dict of wildcards,
  and return a list of strings,
  to handle complex dependency issues that can't be expressed in pure Snakemake.
- Import `glob.glob` to match multiple files on disk matching a specific pattern.
  Don't rely on this finding intermediate files in final production workflows,
  since it won't find files not present at the start of the workflow run.
- Use `snakemake --touch` if you need to mark files as up-to-date,
  so that Snakemake won't try to regenerate them.

::::::::::::::::::::::::::::::::::::::::::::::::::
