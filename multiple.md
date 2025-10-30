---
title: "Multiple inputs and outputs"
teaching: 20
exercises: 15
---

:::::::::::::::::::::::::::::::::::::: questions 

- How do I write rules that require or use more than one file, or class of file?
- How do I tell Snakemake to not delete log files when jobs fail?
- What do Snakemake errors look like, and how do I read them?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Be able to write rules with multiple named inputs and outputs
- Know how and when to specify `log:` within a rule
- Be aware that Snakemake errors are common
- Understand how to approach reading Snakemake errors when they occur

::::::::::::::::::::::::::::::::::::::::::::::::

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
    params:
        plateau_start=lookup(within=metadata, query="beta == {beta}", cols="ps_plateau_start"),
        plateau_end=lookup(within=metadata, query="beta == {beta}", cols="ps_plateau_end"),
    conda: "envs/analysis.yml"
    shell:
        "python -m su2pg_analysis.meson_mass {input} --output_file {output.data} --plateau_start {params.plateau_start} --plateau_end {params.plateau_end} --plot_file {output.plot} --plot_styles {config[plot_styles]}"
```

Rather than having a single string after `output:`,
we now have a block with two lines.
Each line has the format `name=value`,
and is followed by a comma.
To make use of these variables in our rule,
we follow `output` by a `.`,
and then the name of the variable we want to use,
similarly to what we do for `wildcards` and `params`.

Let's test that this works correctly:

```snakemake
snakemake --cores 1 --forceall --printshellcmds --use-conda intermediary_data/beta2.0/corr.ps_eff_mass.pdf
```

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
        plot=multiext(
            "intermediary_data/{subdir}/wflow.W_flow",
            config["plot_filetype"],
	),
    conda: "envs/analysis.yml"
    shell:
        "python -m su2pg_analysis.w0 {input} --W0 {config[W0_reference]} --output_file {output.data} --plot_file {output.plot} --plot_styles {config[plot_styles]}"
```

Again,
we can test this with

```shellsession
snakemake --cores 1 --forceall --printshellcmds --use-conda intermediary_data/beta2.0/wflow.W_flow.pdf
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
# Estimate renormalised decay constant
rule one_loop_matching:
    input:
        plaquette="intermediary_data/{subdir}/pg.plaquette.json.gz",
        meson="intermediary_data/{subdir}/corr.{channel}_mass.json.gz",
    output:
        data="intermediary_data/{subdir}/pg.corr.{channel}_decay_const.json.gz",
    conda: "envs/analysis.yml"
    shell:
        "python -m su2pg_analysis.one_loop_matching --plaquette_data {input.plaquette} --spectral_observable_data {input.meson} --output_filename {output.data}"
```

To test this:

```shellsession
$ snakemake --cores 1 --forceall --printshellcmds --use-conda intermediary_data/beta2.0/pg.corr.ps_decay_const.json.gz
$ cat intermediary_data/beta2.0/pg.corr.ps_decay_const.json.gz | gunzip | head -n 28
```

```output
{
 "program": "pyerrors 2.14.0",
 "version": "1.1",
 "who": "ed",
 "date": "2025-10-08 20:56:18 +0100",
 "host": "azusa, Linux-6.8.0-85-generic-x86_64-with-glibc2.39",
 "description": {
  "INFO": "This JSON file contains a python dictionary that has been parsed to a list of structures. OBSDICT contains the dictionary, where Obs or other structures have been replaced by DICTOBS[0-9]+. The field description contains the additional description of this JSON file. This file may be parsed to a dict with the pyerrors routine load_json_dict.",
  "OBSDICT": {
   "decay_const": "DICTOBS0"
  },
  "description": {
   "group_family": "SU",
   "num_colors": 2,
   "representation": "fun",
   "nt": 48,
   "nx": 24,
   "ny": 24,
   "nz": 24,
   "beta": 2.0,
   "mass": 0.0,
   "channel": "ps"
  }
 },
 "obsdata": [{
   "type": "Obs",
   "layout": "1",
   "value": [0.06760436978217312],
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

Hint:
first,
write a temporary rule to check the output of 

```shellsession
python src/plot_spectrum.py --help
```

(Or otherwise,
create a Conda environment based on `envs/analysis.yml`,
and temporarily activate it to run the command.
Remember to deactivate once you're finished,
since otherwise you will no longer have access to `snakemake`.)

:::::::::::::::  solution

The help output for `plot_spectrum.py` is:

```output
usage: plot_spectrum.py [-h] [--output_filename OUTPUT_FILENAME] [--plot_styles PLOT_STYLES] [--x_observable X_OBSERVABLE]
                        [--y_observable Y_OBSERVABLE] [--zero_x_axis] [--zero_y_axis]
                        datafile [datafile ...]

Plot one state against another for each ensemble

positional arguments:
  datafile              Data files to read and plot

options:
  -h, --help            show this help message and exit
  --output_filename OUTPUT_FILENAME
                        Where to put the plot
  --plot_styles PLOT_STYLES
                        Plot style file to use
  --x_observable X_OBSERVABLE
                        Observables to put on the horizontal axis
  --y_observable Y_OBSERVABLE
                        Observables to put on the vertical axis
  --zero_x_axis         Ensure that zero is present on the vertical axis
  --zero_y_axis         Ensure that zero is present on the vertical axis
```

Based on this,
a possible rule is:

```snakemake
rule spectrum:
    input:
        script="src/plot_spectrum.py",
        ps_mass=expand(
            "intermediary_data/beta{beta}/corr.ps_mass.json.gz",
            beta=[1.5, 1.6, 1.7, 1.8, 1.9, 2.0, 2.1, 2.2, 2.3, 2.4, 2.5],
        ),
        ps_decay_const=expand(
            "intermediary_data/beta{beta}/pg.corr.ps_decay_const.json.gz",
            beta=[1.5, 1.6, 1.7, 1.8, 1.9, 2.0, 2.1, 2.2, 2.3, 2.4, 2.5],
        ),
    output:
        plot=multiext("assets/plots/spectrum", config["plot_filetype"]),
    conda: "envs/analysis.yml"
    shell:
        "python {input.script} {input.ps_mass} {input.ps_decay_const} --y_observable f_ps --zero_y_axis --zero_x_axis --output_file {output.plot} --plot_styles {config[plot_styles]}"
```

Test this using

```shellsession
snakemake --cores 1 --forceall --printshellcmds --use-conda assets/plots/spectrum.pdf
```

:::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::: instructor

## Definitely run through the spectrum plot

This plot is referred to from subsequent lessons,
so you definitely need to go through it.

:::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::  challenge

## Scaled spectrum plot

Write a rule that plots both
the pseudoscalar channel's decay constant and the vector mass
against the pseudoscalar mass,
all scaled by the $w_0$ scale,
for each ensemble studied having $\beta \le 1.8$.
The tool `src/plot_spectrum.py` will help with this.

Hint:
compared to the unscaled spectrum plot,
you will additionally need data files for the vector mass and $w_0$,
and will need to pass the `--rescale_w0` option to `plot_spectrum.py`.

:::::::::::::::  solution

```snakemake
rule spectrum_scaled:
    input:
        script="src/plot_spectrum.py",
        ps_mass=expand(
            "intermediary_data/beta{beta}/corr.ps_mass.json.gz",
            beta=[1.5, 1.6, 1.7, 1.8],
        ),
        v_mass=expand(
            "intermediary_data/beta{beta}/corr.v_mass.json.gz",
            beta=[1.5, 1.6, 1.7, 1.8],
        ),
        ps_decay_const=expand(
            "intermediary_data/beta{beta}/pg.corr.ps_decay_const.json.gz",
            beta=[1.5, 1.6, 1.7, 1.8],
        ),
        w0=expand(
        "intermediary_data/beta{beta}/wflow.w0.json.gz",
            beta=[1.5, 1.6, 1.7, 1.8],
    ),
    output:
        plot=multiext("assets/plots/spectrum_scaled", config["plot_filetype"]),
    conda: "envs/analysis.yml"
    shell:
        "python {input.script} {input.ps_mass} {input.v_mass} {input.ps_decay_const} {input.w0} --y_observable f_ps --y_observable m_v --zero_y_axis --rescale_w0 --output_file {output.plot} --plot_styles {config[plot_styles]}"
```

Test this using

```shellsession
snakemake --cores 1 --forceall --printshellcmds --use-conda assets/plots/spectrum_scaled.pdf
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
        "python -m su2pg_analysis.meson_mass {input} --output_file {output.data} --plateau_start {params.plateau_start} --plateau_end {params.plateau_end} --plot_file {output.plot} --plot_styles {config[plot_styles]} 2>&1 | tee {log.messages}"
```

We can again verify this using

```shellsession
snakemake --cores 1 --forceall --printshellcmds --use-conda intermediary_data/beta2.0/corr.ps_mass.json.gz
cat intermediary_data/beta{beta}/corr.ps_mass.log
```

Since this fit didn't emit any output on this occasion,
the resulting log is empty.

:::::::::::::::::::::::::::::::::::::::::  callout

## `2>&1 | tee`

You may recall that `|` is the pipe operator in the Unix shell,
taking standard output from one program and passing it to standard input of the next.
(If this is unfamiliar,
you may wish to look through
the Software Carpentry introduction to [the Unix shell][shell-novice]
when you have a moment.)

Adding `2>&1` means that
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
        ps_mass=expand(
            "intermediary_data/beta{beta}/corr.ps_mass.json.gz",
            beta=[1.5, 1.6, 1.7, 1.8, 1.9, 2.0, 2.1, 2.2, 2.3, 2.4, 2.5],
        ),
        ps_decay_const=expand(
            "intermediary_data/beta{beta}/pg.corr.ps_decay_const.json.gz",
            beta=[1.5, 1.6, 1.7, 1.8, 1.9, 2.0, 2.1, 2.2, 2.3, 2.4, 2.5],
        ),
    output:
        plot=multiext("assets/plots/spectrum", config["plot_filetype"]),
    log:
        messages="intermediary_data/spectrum_plot.log"
    conda: "envs/analysis.yml"
    shell:
        "python {input.script} {input.ps_mass} {input.ps_decay_const} --y_observable f_ps --zero_y_axis --zero_x_axis --output_file {output.plot} --plot_styles {config[plot_styles]} 2>&1 | tee {log.messages}"
```

:::::::::::::::::::::::::


We can again verify this using

```shellsession
snakemake --cores 1 --forceall --printshellcmds --use-conda assets/plots/spectrum.pdf
cat intermediary_data/spectrum_plot.log
```

::::::::::::::::::::::::::::::::::::::::::::::::::

## Dealing with errors

We'll end the episode by looking at a common problem that can arise
if you mistype a filename in a rule.
It may seem silly to break the workflow when we just got it working,
but it will be instructive,
so let's modify the Snakefile and deliberately specify an incorrect output filename
in the `ps_mass` rule.

```snakemake
...
shell:
        "python -m su2pg_analysis.meson_mass {input} --output_file {output.data}.json_ --plateau_start {params.plateau_start} --plateau_end {params.plateau_end} --plot_file {output.plot} --plot_styles {config[plot_styles]} 2>&1 | tee {log.messages}"
```

To keep things tidy,
this time we'll manually remove the intermediary data directory.

```bash
$ rm -rvf intermediary_data
```

And re-run.

```shellsession
$ snakemake --cores 1 --forceall --printshellcmds --use-conda intermediary_data/beta2.0/corr.ps_mass.json.gz
Assuming unrestricted shared filesystem usage.
host: azusa
Building DAG of jobs...
Using shell: /usr/bin/bash
Provided cores: 1 (use --cores to define parallelism)
Rules claiming more threads will be scaled down.
Job stats:
job        count
-------  -------
ps_mass        1
total          1

Select jobs to execute...
Execute 1 jobs...

[Tue Sep  2 00:29:00 2025]
localrule ps_mass:
    input: raw_data/beta2.0/out_corr
    output: intermediary_data/beta2.0/corr.ps_mass.json.gz, intermediary_data/beta2.0/corr.ps_eff_mass.pdf
    log: intermediary_data/beta2.0/corr.ps_mass.log
    jobid: 0
    reason: Forced execution
    wildcards: beta=2.0
    resources: tmpdir=/tmp
Shell command: python -m su2pg_analysis.meson_mass raw_data/beta2.0/out_corr --output_file intermediary_data/beta2.0/corr.ps_mass.json.gz.json_ --plateau_start 11 --plateau_end 21 --plot_file intermediary_data/beta2.0/corr.ps_eff_mass.pdf --plot_styles styles/prd.mplstyle 2>&1 | tee intermediary_data/beta2.0/corr.ps_mass.log
Activating conda environment: .snakemake/conda/7974a14bb2d9244fc9da6963ef6ee6d6_
Waiting at most 5 seconds for missing files:
intermediary_data/beta2.0/corr.ps_mass.json.gz (missing locally)
MissingOutputException in rule ps_mass in file "/home/ed/src/su2pg/workflow/Snakefile", line 68:
Job 0 completed successfully, but some output files are missing. Missing files after 5 seconds. This might be due to filesystem latency. If that is the case, consider to increase the wait time with --latency-wait:
intermediary_data/beta2.0/corr.ps_mass.json.gz (missing locally, parent dir contents: corr.ps_mass.json.gz.json_.json.gz, corr.ps_mass.log, corr.ps_eff_mass.pdf)
Removing output files of failed job ps_mass since they might be corrupted:
intermediary_data/beta2.0/corr.ps_eff_mass.pdf
Shutting down, this might take some time.
Exiting because a job execution failed. Look below for error messages
[Tue Sep  2 00:29:17 2025]
Error in rule ps_mass:
    message: None
    jobid: 0
    input: raw_data/beta2.0/out_corr
    output: intermediary_data/beta2.0/corr.ps_mass.json.gz, intermediary_data/beta2.0/corr.ps_eff_mass.pdf
    log: intermediary_data/beta2.0/corr.ps_mass.log (check log file(s) for error details)
    conda-env: /home/ed/src/su2pg/.snakemake/conda/7974a14bb2d9244fc9da6963ef6ee6d6_
    shell:
        python -m su2pg_analysis.meson_mass raw_data/beta2.0/out_corr --output_file intermediary_data/beta2.0/corr.ps_mass.json.gz.json_ --plateau_start 11 --plateau_end 21 --plot_file intermediary_data/beta2.0/corr.ps_eff_mass.pdf --plot_styles styles/prd.mplstyle 2>&1 | tee intermediary_data/beta2.0/corr.ps_mass.log
        (command exited with non-zero exit code)
Complete log(s): /home/ed/src/su2pg/.snakemake/log/2025-09-02T002859.356054.snakemake.log
WorkflowError:
At least one job did not complete successfully.
```

There's a lot to take in here.
Some of the messages are very informative.
Some less so.

1. Snakemake did actually run the tool,
   as evidenced by the output from the program that we see on the screen.
2. Python is reporting that there is a file missing.
3. Snakemake complains one expected output file is missing:
   `intermediary_data/beta2.0/corr.ps_mass.json.gz`.
4. The other expected output file
   `intermediary_data/beta2.0/corr.ps_eff_mass.pdf`
   was found but has now been removed by Snakemake.
5. Snakemake suggests this might be due to "filesystem latency".

This last point is a red herring.
"Filesystem latency" is not an issue here, and never will be,
since we are not using a network filesystem.
We know what the problem is,
as we deliberately caused it,
but to diagnose an unexpected error like this we would investigate further
by looking at the `intermediary_data/beta2.0` subdirectory.

```shellsession
$ ls intermediary_data/beta2.0/
corr.ps_mass.json.gz.json_.json.gz  corr.ps_mass.log
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
snakemake --cores 1 --forceall --printshellcmds --use-conda assets/plots/spectrum.pdf
```

:::::::::::::::::::::::::::::::::::::::: keypoints

- Rules can have multiple inputs and outputs, separated by commas
- Use `name=value` to give names to inputs/outputs
- Inputs themselves can be lists
- Use placeholders like `{input.name}` to refer to single named inputs
- Where there are multiple inputs, `{input}` will insert them all, separated by spaces
- Use `log:` to list log outputs, which will not be removed when jobs fail
- Errors are an expected part developing Snakemake workflows,
  and usually give enough information to track down what is causing them

::::::::::::::::::::::::::::::::::::::::::::::::::


[shell-novice]: https://swcarpentry.github.io/shell-novice
