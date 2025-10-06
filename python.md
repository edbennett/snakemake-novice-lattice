---
title: "Running Python code with Snakemake"
teaching: 10
exercises: 5
---

:::::::::::::::::::::::::::::::::::::: questions 

- How do I configure an environment to run Python with Snakemake?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Create a Conda environment definition
- Use Snakemake to instantiate this environment and use it to run Python code

::::::::::::::::::::::::::::::::::::::::::::::::

## Why define an environment

Snakemake is written in Python,
so anyone running a Snakemake workflow already has Python available.
In principle,
we could make use of this installation to run
any Python code we need to run in our workflow.
However,
it's more than likely we will need to make use of libraries
that are not installed as part of the Snakemake installation.
At that point,
we would either need to install additional libraries into our Snakemake environment,
which might be used by other analyses that need their own sets of libraries
that could create conflicts,
or to create a second Snakemake environment for this analysis.
If different steps of our workflow need different, conflicting sets of libraries
then this becomes more complicated again.

We would also like those trying to reproduce our work
to be able to run using exactly the same software environment
that we used in our original work.
In principle,
we could write detailed documentation specifying which packages to install;
however,
it is both more precise and more convenient to define the environment as a data file,
which Conda can use to build the same environment every time.

Even better,
we can tell Snakemake to use a specific Conda environment for each rule we define.

## A basic environment definition

Conda environment definitions are created in [YAML][yaml]-format files.
These specify what Conda packages are needed
(including the target version of Python),
as well as any Pip packages that are installed.

:::::::::::::::::::::::::::::::::::::::  callout

Some packages give you a choice as to whether to install using Conda or Pip.
When working interactively with an environment,
using Pip consistently typically reduces the chance of
getting into a state where Conda is unable to install packages.
That is less of a problem when constructing new environments from definition files,
but even so,
using Pip where possible will typically
allow environments to resolve and install more quickly.

::::::::::::::::::::::::::::::::::::::::::::::::::

By convention,
Conda environment definitions in Snakemake workflows are placed in a
`workflow/envs/` directory.
Let's create `workflow/envs/analysis.yml` now,
and place the following content into it

```yaml
name: su2pg_analysis
channels:
  - conda-forge
dependencies:
  - pip=24.2
  - python=3.12.6
  - pip:
      - h5py==3.11.0
      - matplotlib==3.9.2
      - numpy==2.1.1
      - pandas==2.2.3
      - scipy==1.14.1
      - uncertainties==3.2.2
      - corrfitter==8.2
      - -e ../../libs/su2pg_analysis
```

This will install the specified versions of
h5py, Matplotlib, Numpy, Pandas, Scipy, uncertainties, and corrfitter,
as well as the analysis tools provided in the `libs` directory.
The latter are installed in editable mode,
so if you need to modify them
to fix bugs or add functionality while working on your workflow,
you don't need to remember to manually reinstall them.

## Using an environment definition in a Snakefile

Now that we have created an environment file,
we can use it in our Snakefile
to compute the average plaquette from a configuration generation log.
Let's add the following rule to `workflow/Snakefile`:

```snakemake
rule avg_plaquette:
    input: "raw_data/beta2.0/out_pg"
    output: "intermediary_data/beta2.0/pg.plaquette.json.gz"
    conda: "envs/analysis.yml"
    shell:
        "python -m su2pg_analysis.plaquette raw_data/beta2.0/out_pg --output_file intermediary_data/beta2.0/pg.plaquette.json.gz"
```

The `conda:` block tells Snakemake where to find
the Conda environment that should be used for running this rule.
Let's test this now:

```shellsession
snakemake --cores 1 --forceall --printshellcmds --use-conda intermediary_data/beta2.0/pg.plaquette.json.gz
```

We need to specify `--use-conda`
to tell Snakemake to pay attention to the `conda:` specification.

Let's check now that the output was correctly generated:

```shellsession
$ cat intermediary_data/beta2.0/pg.plaquette.json.gz | gunzip | head -n 20
{
 "program": "pyerrors 2.13.0",
 "version": "1.1",
 "who": "ed",
 "date": "2025-05-20 15:44:09 +0100",
 "host": "tsukasa.lan, macOS-15.4-arm64-arm-64bit",
 "description": {
  "group_family": "SU",
  "num_colors": 2,
  "nt": 48,
  "nx": 24,
  "ny": 24,
  "nz": 24,
  "beta": 2.0,
  "num_heatbath": 1,
  "num_overrelaxed": 4,
  "num_thermalization": 1000,
  "thermalization_time": 2453.811479,
  "num_trajectories": 10010
 },
```

Some of the output will differ on your machine,
since this library tracks provenance,
such as where and when the code was run,
in the output file.

:::::::::::::::::::::::::::::::::::::::  callout

You might notice that this output contains
a lot of information besides the average plaquette.
These are metadata&mdash;that is,
data describing the data,
which help us understand it and make better use of it.
This includes physics parameters describing what the data refer to,
and _provenance_ information describing how and when it was computed.

If you imagine a script that outputs only the average plaquette and its uncertainty:

```
0.501206452535401 TODO
```

then seeing just this file in isolation,
it would be much harder to understand what it means or where it comes from.
You would need some other bookkeeping system to track the physics parameters.
Bundling these into the files means we are less likely
to accidentally create a situation
where the data are presented with incorrect labels.

We're using JSON format to output the results;
if you are not using a library that automatically generates JSON,
you might instead use CSV or any other format.
The important part is that it can be read and written easily,
and can hold the metadata that you need to keep.

We compress each output file with GZIP (the `.gz` extension),
because the `pyerrors` library that we are using does this automatically.
Since we are generating one output file per computation we're performing,
we'll end up with a lot of files;
each has a lot of metadata within it,
so this might take up a lot of space without compression.

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::  challenge

## More plaquettes

Add a second rule to compute the average plaquette in the file
`intermediary_data/beta2.2/out_pg`.
Add this to the same Snakefile you already made,
under the `avg_plaquette` rule,
and run your rules in the terminal.
When running the `snakemake` command
you'll need to tell Snakemake to make both the output files.

:::::::::::::::  solution

## Solution

You can choose whatever name you like for this second rule,
but it can't be `avg_plaquette`,
as rule names need to be unique within a Snakefile.
So in this example answer we use `avg_plaquette2`.


```snakemake
rule avg_plaquette2:
    input: "raw_data/beta2.2/out_pg"
    output: "intermediary_data/beta2.2/pg.plaquette.json.gz"
    conda: "envs/analysis.yml"
    shell:
        "python -m su2pg_analysis.plaquette raw_data/beta2.2/out_pg --output_file intermediary_data/beta2.2/pg.plaquette.json.gz"
```

Then in the shell:

```shellsession
snakemake --cores 1 --forceall --printshellcmds --use-conda intermediary_data/beta2.0/pg.plaquette.json.gz intermediary_data/beta2.2/pg.plaquette.json.gz
```

If you think writing a separate rule for each output file is silly, you are correct.
We'll address this next!

:::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::


[yaml]: https://yaml.org
