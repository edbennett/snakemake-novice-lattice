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
Let's add the following rule to `envs/Snakefile`:

```snakemake
rule avg_plaquette:
    input: "raw_data/beta4.0/out_pg"
    output: "intermediary_data/beta4.0/pg.plaquette.json"
    conda: "../envs/analysis.yml"
    shell:
        "python -m su2pg_analysis.plaquette raw_data/beta4.0/out_pg --output_file intermediary_data/beta4.0/pg.plaquette.json"
```

The `conda:` block tells Snakemake where to find
the Conda environment that should be used for running this rule.
Let's test this now:

```shellsession
snakemake --jobs 1 --forceall --printshellcmds --use-conda intermediary_data/beta4.0/pg.plaquette.json
```

We need to specify `--use-conda`
to tell Snakemake to pay attention to the `conda:` specification.

Let's check now that the output was correctly generated:

```shellsession
$ cat intermediary_data/beta4.0/pg.plaquette.json
TODO
```

Some of the output will differ on your machine,
since this library tracks provenance,
such as where and when the code was run,
in the output file.

:::::::::::::::::::::::::::::::::::::::  challenge

## More plaquettes

Add a second rule to compute the average plaquette in the file
`intermediary_data/beta4.5/out_pg`.
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
    input: "raw_data/beta4.5/out_pg"
    output: "intermediary_data/beta4.5/pg.plaquette.json"
    conda: "../envs/analysis.yml"
    shell:
        "python -m su2pg_analysis.plaquette raw_data/beta4.5/out_pg --output_file intermediary_data/beta4.5/pg.plaquette.json"
```

Then in the shell:

```shellsession
snakemake --jobs 1 --forceall --printshellcmds --use-conda intermediary_data/beta4.0/pg.plaquette.json intermediary_data/beta4.5/pg.plaquette.json
```

If you think writing a separate rule for each output file is silly, you are correct.
We'll address this next!

:::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::


[yaml]: https://yaml.org
