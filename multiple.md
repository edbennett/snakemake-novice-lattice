---
title: "Multiple inputs and outputs"
teaching: 10
exercises: 5
---

## Multiple outputs

Quite frequently,
we will want a rule to be able to generate more than one file.
It's important we let Snakemake know about this,
both so that it can instruct our tools on where to place these files,
and so it can verify that they are correctly created by the rule.
For example,
when fitting a correlation function with a plateau region that we specify,
it's important to look at an effective mass plot
to verify that the plateau actually matches what we assert.
The rule we just wrote doesn't do this&mdash;it
only spits out a numerical answer.
Let's update this rule so that it can also generate the effective mass plot.

```snakemake
rule ps_mass:
    input: "raw_data/beta{beta}/out_corr"
    output:
        data="intermediary_data/beta{beta}/corr.ps_mass.json.gz",
        plot="intermediary_data/beta{beta}/corr.ps_eff_mass.{plot_filetype}",
    params:
        plateau_start: lookup(within=metadata, query="beta = {beta}", cols="ps_plateau_start"),
        plateau_end: lookup(within=metadata, query="beta = {beta}", cols="ps_plateau_end"),
    conda: "envs/analysis.yml"
    shell:
        "python -m su2pg_analysis.meson_mass {input} --output_file {output.data} --plateau_start {params.plateau_start} --plateau_end {params.plateau_end} --plot_file {output.plot} --plot_styles {plot_styles}"
```

Rather than having a single string after `output:`,
we now have a block with two lines.
Each line has the format `name=value`,
and is followed by a comma.
To make use of these variables in our rule,
we follow `output` by a `.`,
and then the name of the variable we want to use,
similarly to what we do for `wildcards` and `params`.

:::::::::::::::::::::::::::::::::::::::  challenge

## Non-specificity

What happens if we define multiple named `output:` variables like this,
but refer to the `{output}` placeholder in the `shell:` block
without specifying a variable name?

(One way to find this out is to try
`echo {output}` as the entire `shell:` content;
this will generate a missing output error,
but will first let you see what the output is.)

:::::::::::::::  solution

Snakemake will provide all of the defined output variables,
as a space-separated list.
This is similar to what happens when an output variable is a list,
as we saw earlier when looking at the `expand()` function.

:::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::  challenge

## Flow plots

Update the Wilson flow $w_0$ computation that we looked at in a previous challenge
to also output the flow of $\mathcal{W}(t)$,
so that the shape of the flow may be checked.

:::::::::::::::  solution

```snakemake
rule w0:
    input: "raw_data/{subdir}/out_wflow"
    output: 
        data="intermediary_data/{subdir}/wflow.w0.json.gz",
        plot="intermediary_data/{subdir}/wflow.W_flow.{plot_filetype}",
    conda: "envs/analysis.yml"
    shell:
        "python -m su2pg_analysis.w0 {input} --W0 {W0_reference} --output_file {output} --plot_file {output.plot} --plot_styles {plot_styles}"
```

:::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::

## Multiple inputs

Similarly to outputs,
there are many situations
where we want to work with more than one class of input file&mdash;for example,
to combine differing observables into one.
For example,
the `meson_mass` rule we wrote previously
also outputs the amplitude of the exponential.
When combined with the average plaquette via one-loop matching,
this can be used to give an estimate of the decay constant.
The syntax for this is the same as we saw above for `output:`.

```snakemake
rule one_loop_matching:
    input:
        plaquette="intermediary_data/{subdir}/pg.plaquette.json.gz",
        meson="intermediary_data/{subdir}/corr.{channel}_mass.json.gz",
    output:
        data="intermediary_data/{subdir}/pg.corr.{channel}_decay_const.json.gz",
    conda: "envs/analysis.yml"
    shell:
        "python -m su2pg_analysis.one_loop_matching --plaquette {input.plaquette} --meson {input.meson} --output_file {output.data}"
```

:::::::::::::::::::::::::::::::::::::::::  callout

## Naming things

Even when there is only one `output:` file,
we are still allowed to name it.
This makes life easier if we need to add more outputs later,
and can make it a little clearer what our intent is
when we come to read the workflow later.

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::  challenge

## Spectrum plot

Write a rule that plots the pseudoscalar channel's decay constant against its mass,
for each ensemble studied.
The tool `src/plot_spectrum.py` will help with this.

Try making the filename of the tool a parameter too,
so that if the script is changed,
Snakemake will correctly re-run the workflow.

:::::::::::::::  solution

```snakemake
rule spectrum:
    input:
        script="src/plot_spectrum.py",
        ps_mass="intermediary_data/{subdir}/corr.{channel}_mass.json.gz",
        ps_decay_const="intermediary_data/{subdir}/pg.corr.{channel}_decay_const.json.gz",
    output:
        plot="assets/plots/spectrum.{plot_filetype}"
    conda: "envs/analysis.yml"
    shell:
        "python {input.script} {input.ps_mass} {input.ps_decay_const} --output_file {output.plot} --plot_styles {plot_styles}"
```

:::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::

## Log files

When a process run by Snakemake exits with an error code,
Snakemake removes all the expected output files.
Usually this is what we want:
we don't want to have potentially corrupt output,
that might be used as input for subsequent rules.
However,
there are some classes of output file that are useful in helping to identify
what caused the error in the first place:
log files.

We can tell Snakemake that specified files are log files,
rather than regular output files,
by placing them in a `log:` block rather than an `output:` one.
Snakemake will not delete a file marked as a log
if an error is raised by the process generating it.

For example,
for the `ps_mass` rule above,
we might use:

```snakemake
rule ps_mass:
    input: 
        data="raw_data/beta{beta}/out_corr",
    output:
        data="intermediary_data/beta{beta}/corr.ps_mass.json.gz",
        plot="intermediary_data/beta{beta}/corr.ps_eff_mass.{plot_filetype}",
    log:
        messages="intermediary_data/beta{beta}/corr.ps_mass.log",
    params:
        plateau_start: lookup(within=metadata, query="beta = {beta}", cols="ps_plateau_start"),
        plateau_end: lookup(within=metadata, query="beta = {beta}", cols="ps_plateau_end"),
    conda: "envs/analysis.yml"
    shell:
        "python -m su2pg_analysis.meson_mass {input.data} --output_file {output.data} --plateau_start {params.plateau_start} --plateau_end {params.plateau_end} --plot_file {output.plot} --plot_styles {plot_styles} |& tee {log.messages}"
```

:::::::::::::::::::::::::::::::::::::::::  callout

## `|& tee`

You may recall that `|` is the pipe operator in the Unix shell,
taking standard output from one program and passing it to standard input of the next.
(If this is unfamiliar,
you may wish to look through
the Software Carpentry introduction to [the Unix shell][shell-novice]
when you have a moment.)

Adding the `&` symbol means that
both the standard output and standard error streams are piped,
rather than only standard output.
This is useful for a log file,
since we will typically want to see errors there.

The `tee` command "splits a pipe";
that is,
it takes standard input
and outputs it both to standard output and to the specified filename.
This way,
we get the log on disk,
but also output to screen as well,
so we can monitor issues as the workflow runs.

::::::::::::::::::::::::::::::::::::::::::::::::::


:::::::::::::::::::::::::::::::::::::::  challenge

## Logged plots

Adjust the solution for plotting the spectrum above
so that any warnings or errors generated by the plotting script
are logged to a file.

:::::::::::::::  solution

```snakemake
rule spectrum:
    input:
        script="src/plot_spectrum.py",
        ps_mass="intermediary_data/{subdir}/corr.{channel}_mass.json.gz",
        ps_decay_const="intermediary_data/{subdir}/pg.corr.{channel}_decay_const.json.gz",
    output:
        plot="assets/plots/spectrum.{plot_filetype}"
    log:
        messages="intermediary_data/{subdir}/pg.corr.{channel}_mass.log"
    conda: "envs/analysis.yml"
    shell:
        "python {input.script} {input.ps_mass} {input.ps_decay_const} --output_file {output.plot} --plot_styles {plot_styles} |& tee {log.messages}"
```

:::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::

## Dealing with errors

We'll end the chapter by looking at a common problem that can arise
if you mistype a filename in a rule.
It may seem silly to break the workflow when we just got it working,
but it will be instructive,
so let's modify the Snakefile and deliberately specify an incorrect output filename
in the `ps_mass` rule.

```snakemake
...
shell:
        "python -m su2pg_analysis.meson_mass {input.data} --output_file {output.data}.json_ --plateau_start {params.plateau_start} --plateau_end {params.plateau_end} --plot_file {output.plot} --plot_styles {plot_styles} |& tee {log.messages}"
```

To keep things tidy,
this time we'll manually remove the intermediary data directory.

```bash
$ rm -rvf intermediary_data
```

And re-run.

```shellsession
$ snakemake --jobs 1 --forceall --printshellcmds --use-conda intermediary_data/beta4.0/corr.ps_mass.json

...
TODO
```

There's a lot to take in here.
Some of the messages are very informative.
Some less so.

1. Snakemake did actually run the tool,
   as evidenced by the output from the program that we see on the screen.
2. Python is reporting that there is a file missing.
3. Snakemake complains one expected output file is missing:
   `intermediary_data/beta4.0/corr.ps_mass.json`.
4. The other expected output file
   `intermediary_data/beta4.0/corr.ps_eff_mass.pdf`
   was found but has now been removed by Snakemake.
5. Snakemake suggest this might be due to "filesystem latency".

This last point is a red herring.
"Filesystem latency" is not an issue here, and never will be,
since we are not using a network filesystem.
We know what the problem is,
as we deliberately caused it,
but to diagnose an unexpected error like this we would investigate further
by looking at the `intermediary_data/beta4.0` subdirectory.

```shellsession
$ ls intermediary_data/beta4.0/
TODO
```

Remember that Snakemake itself does not create any output files.
It just runs the commands you put in the `shell` sections,
then checks to see if all the expected output files have appeared.

So if the file names created by your rule are not exactly the same
as in the `output:` block
you will get this error,
and you will,
in this case,
find that some output files are present but others
(`corr.ps_eff_mass.pdf`, which was named correctly)
have been cleaned up by Snakemake.

:::::::::::::::::::::::::::::::::::::::::  callout

## Errors are normal

Don't be disheartened if you see errors like the one above
when first testing your new Snakemake workflows.
There is a lot that can go wrong when writing a new workflow,
and you'll normally need several iterations to get things just right.
One advantage of the Snakemake approach compared to regular scripts
is that Snakemake fails fast when there is a problem,
rather than ploughing on
and potentially running junk calculations on partial or corrupted data. Another advantage is that
when a step fails we can safely resume from where we left off,
as we'll see in the next episode.

::::::::::::::::::::::::::::::::::::::::::::::::::

Finally, edit the names in the Snakefile back to the correct version
and re-run to confirm that all is well.

```snakemake
snakemake --jobs 1 --forceall --printshellcmds --use-conda assets/plots/spectrum.pdf
```
