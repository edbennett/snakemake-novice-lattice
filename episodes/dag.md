---
title: "How Snakemake plans jobs"
teaching: 10
exercises: 5
---

::::::::::::::::::::::::::::::::::::::: objectives

- View the DAG for our pipeline
- Understand the logic Snakemake uses when running and re-running jobs

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::: questions

- How do I visualise a Snakemake workflow?
- How does Snakemake avoid unecessary work?
- How do I control what steps will be run?

::::::::::::::::::::::::::::::::::::::::::::::::::

## The DAG

You may have noticed that one of the messages Snakemake always prints is:

```output
Building DAG of jobs...
```

A DAG is a **Directed Acyclic Graph** and it can be pictured like so:

![TODO][fig-dag]

The above DAG is based on three of our existing rules,
and shows all the jobs Snakemake would run
to compute the pseudoscalar decay constant of the $\beta = 4.0$ ensemble.

:::::::::::::::::::::::::::::::::::::::  checklist

## Note that:

- A rule can appear more than once,
  with different wildcards
  (a **rule** plus **wildcard values** defines a **job**)
- A rule may not be used at all,
  if it is not required for the target outputs
- The arrows show dependency ordering between jobs
- Snakemake can run the jobs in any order that doesn't break dependency.
  For example,
  *one_loop_matching* cannot run until
  both *ps_mass* and *avg_plaquette* have completed,
  but it may run before or after *count_trajectories*
- This is a work list,
  *not a flowchart*,
  so there are no if/else decisions or loops.
  Snakemake runs every job in the DAG exactly once
- The DAG depends both on the Snakefile *and* on the requested target outputs,
  and the files already present
- When building the DAG,
  Snakemake does not look at the *shell* part of the rules at all.
  Only when running the DAG will Snakemake check that
  the shell commands are working and producing the expected output files

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::  challenge

## How many jobs?

If we asked Snakemake to run `one_loop_matching` on all twelve ensembles
(`beta2.0` to `beta10.0`),
how many jobs would that be in total?

:::::::::::::::  solution

## Solution

36 in total:

- 12 $\times$ `one_loop_matching` 
- 12 $\times$ `ps_mass`
- 12 $\times$ `avg_plaquette`
- 0 $\times$ `count_trajectories`

:::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::

## Snakemake is lazy, and laziness is good

For the last few episodes, we've told you to run Snakemake like this:

```shellsession
snakemake --jobs 1 --forceall --printshellcmds --use-conda 
```

As a reminder,
the `--jobs 1` flag tells Snakemake to run one job at a time,
`--printshellcmds` is to print out the shell commands before running them,
and `--use-conda` to ensure that Snakemake sets up the correct Conda environment.

The `--forceall` flag turns on `forceall` mode,
and in normal usage you don't want this.

At the end of the last chapter,
we generated a spectrum plot by running:

```shellsession
snakemake --jobs 1 --forceall --printshellcmds --use-conda assets/plots/spectrum.pdf
```

Now try without the `--forceall` option.
Assuming that the output files are already created,
you'll see this:

```shellsession
$ snakemake --jobs 1 --forceall --printshellcmds --use-conda assets/plots/spectrum.pdf
TODO
```

In normal operation,
Snakemake only runs a job if:

1. A target file you explicitly requested to make is missing,
1. An intermediate file is missing
   and it is needed in the process of making a target file,
1. Snakemake can see an input file which is newer than an output file,
   or
1. A rule definition or configuration has changed since the output file was created.

The last of these relies on
a ledger that Snakemake saves into the `.snakemake` directory.

Let's demonstrate each of these in turn,
by altering some files and re-running Snakemake without the `--forceall` option.

```bash
$ rm assets/plots/spectrum.pdf
$ snakemake --jobs 1 --printshellcmds --use-conda assets/plots/spectrum.pdf
```

This just re-runs `spectrum`,
the final step.

```bash
$ rm intermediary_data/beta*/corr.ps_mass.json
$ snakemake --jobs 1 --printshellcmds --use-conda assets/plots/spectrum.pdf
```

"Nothing to be done".
Some intermediate output is missing,
but Snakemake already has the file you are telling it to make,
so it doesn't worry.

```bash
touch raw_data/beta*/out_pg
snakemake --jobs 1 --printshellcmds --use-conda assets/plots/spectrum.pdf
```

The `touch` command is a standard Unix command that resets the timestamp of the file,
so now the correlators look to Snakemake as if they were just modified.

Snakemake sees that some of the input files used in the process of producing `assets/plots/spectrum.pdf`
are newer than the existing output file,
so it needs to run the `avg_plaquette` and `one_loop_matching` steps again.
Of course,
the `one_loop_matching` step needs the pseudoscalar mass data that we deleted earlier,
so now the correlation function fitting step is re-run also.

## Explicitly telling Snakemake what to re-run

The default timestamp-based logic is really useful when you want to:

1. Change or add some inputs to an existing analysis without re-processing everything
2. Continue running a workflow that failed part-way

In most cases you can also rely on Snakemake to detect when you have edited a rule, 
but sometimes you need to be explicit,
for example 
if you have updated an external script or changed a setting that Snakemake doesn't see.

The `--forcerun` flag allows you to explicitly tell Snakemake that a rule has changed 
and that all outputs from that rule need to be re-evaluated.

```shellsession
snakemake  --forcerun spectrum --jobs 1--printshellcmds --use-conda assets/plots/spectrum.pdf
```

:::::::::::::::::::::::::::::::::::::::::  callout

## Note on `--forcerun`

Due to a quirk of the way Snakemake parses command-line options,
you need to make sure there are options after the `--forcerun ...`,
before the list of target outputs.
If you don't do this,
Snakemake will think that the target files are instead items to add to the `--forcerun` list,
and then when building the DAG it will just try to run the default rule.

The easiest way is to put the `--jobs` flag before the target outputs.
Then you can list multiple rules to re-run,
and also multiple targets,
and Snakemake can tell which is which.

```bash
snakemake --forcerun avg_plaquette ps_mass --jobs 1 --printshellcmds --use-conda intermediary_data/beta2.0/pg.corr.ps_decay_const.json intermediary_data/beta2.5/pg.corr.ps_decay_const.json
```

The reason for using the `--jobs` flag specifically
is that you pretty much always want this option.

::::::::::::::::::::::::::::::::::::::::::::::::::

The `--force` flag specifies that
the target outputs named on the command line should always be regenerated, so you can use this to explicitly re-make specific files.

```bash
$ snakemake --jobs 1 --force --printshellcmds assets/plots/spectrum.pdf
```

This always re-runs `spectrum`,
regardless of whether the output file is there already.
For all intermediate outputs,
Snakemake applies the default timestamp-based logic.
Contrast with `--forceall`,
which runs the entire DAG every time.

## Visualising the DAG

Snakemake can draw a picture of the DAG for you, if you run it like this:

```shellsession
snakemake --force --dag assets/plots/spectrum.pdf | gm display -
```

Using the `--dag` option implicitly activates the `--dry-run` option
so that Snakemake will not actually run any jobs,
it will just print the DAG and stop.
Snakemake prints the DAG in a text format,
so we use the `gm` command to make this into a picture and show it on the screen.

:::::::::::::::::::::::::::::::::::::::::  callout

## Note on `gm display`

The `gm` command is provided by
the [GraphicsMagick toolkit](https://www.graphicsmagick.org/).
On systems where `gm` will not display an image directly,
you can instead save it to a PNG file.
You will need the `dot` program from the [GraphViz package](https://graphviz.org/) installed.

```shellsession
snakemake --force --dag assets/plots/spectrum.pdf | dot -Tpng > dag.png
```

::::::::::::::::::::::::::::::::::::::::::::::::::

![TODO][fig-dag2]

The boxes drawn with dotted lines indicate steps that are not to be run,
as the output files are already present and newer than the input files.

:::::::::::::::::::::::::::::::::::::::  challenge

## Visualising the effect of the `--forcerun` and `--force` flags

Run `one_loop_matching` on the `beta2.0` ensemble,
and then use the `--dag` option as shown above to check:

1) How many jobs will run if you ask again to create this output
   with no `--force`, `--forcerun` or `-forceall` options?

2) How many if you use the `--force` option?

3) How many if you use the `--forcerun ps_mass` option?

4) How many if you edit the metadata file
   so that the `ps_plateau_start` for the $\beta=4.0$ ensemble is `TODO`,
   rather than `TODO`?

:::::::::::::::::::: solution

## Solution

This is a way to make the result in the first place:

```bash
$ snakemake --jobs 1 --printshellcmds intermediary_data/beta2.0/pg.corr.ps_decay_const.json
```

1) This command should show three boxes,
   but all are dotted so no jobs are actually to be run.

```bash
$ snakemake --dag intermediary_data/beta2.0/pg.corr.ps_decay_const.json | gm display -
```

2) The `--force` flag re-runs only the job to create the output file,
   so in this case one box is solid,
   and only that job will run.

3) With `--forcerun ps_mass`,
   the `ps_mass` job will re-run,
   and Snakemake sees that this also requires re-running `one_loop_matching`,
   so the answer is 2.

   If you see a message like the one below,
   it's because you need to put an option after `ps_mass`
   or else Snakemake gets confused about what are parameters of `--forcerun`,
   and what things are targets.

   ```error
   WorkflowError:
   Target rules may not contain wildcards.
   ```

4) Editing the Snakefile has the same effect as
   forcing the `ps_mass` rule to re-run,
   so again there will be two jobs to be run from the DAG.

With older versions of Snakemake this would not be auto-detected,
and in fact you can see this behaviour if you remove the hidden `.snakemake` directory.
Now Snakemake has no memory of the rule change
so it will not re-run any jobs unless explicitly told to.

:::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::  callout

## Removing files to trigger reprocessing

In general, getting Snakemake to re-run things by removing files is a bad idea,
because it's easy to forget about intermediate files
that actually contain stale results and need to be updated.
Using the `--forceall` flag is simpler and more reliable.
If in doubt,
and if it will not be too time consuming,
keep it simple and just use `--forceall` to run the whole workflow from scratch.

For the opposite case where you want to avoid re-running particular steps,
see the `‑‑touch` option of Snakemake mentioned [later in the lesson
](TODO).

::::::::::::::::::::::::::::::::::::::::::::::::::



[fig-dag]: fig/dag_1.svg {alt='
  TODO
  Diagram showing jobs as coloured boxes joined by arrows representing
data flow.
  The box labelled as kallisto\_index is in green at the top,
  with two blue boxes labelled trimreads and two yellow boxes labelled countreads.
  The blue trimreads boxes have arrows into the respective yellow countreads boxes.
  Finally there is a kallisto\_quant job shown as a red box,
  with
incoming arrows from both the trimreads box as well as the kallisto\_index box.'
}
[fig-dag2]: fig/dag_2.png {alt='
  TODO
  A DAG for the partial workflow with four boxes,
  representing two trimreads jobs and a kallisto\_index job,
  then a kallisto\_quant job receiving input from the previous three.
  The boxes for the kallisto\_index and trimreads jobs are dotted,
  but the kallisto\_quant box is solid.'}


:::::::::::::::::::::::::::::::::::::::: keypoints

- A **job** in Snakemake is a **rule** plus **wildcard values**
  (determined by working back from the requested output)
- Snakemake plans its work by arranging all the jobs into a **DAG**
  (directed acyclic graph)
- If output files already exist,
  Snakemake can skip parts of the DAG
- Snakemake compares file timestamps and a log of previous runs
  to determine what need regenerating

::::::::::::::::::::::::::::::::::::::::::::::::::

