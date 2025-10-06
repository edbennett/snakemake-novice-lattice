---
title: Tidying up
teaching: 20
exercises: 20
---

::::::::::::::::::::::::::::::::::::::: objectives

- Be able to avoid having long monolithic Snakefiles
- Be able to have Snakemake generate a set of targets without explicitly specifying them
- Understand how to effectively use Git to version control Snakemake worfklows
- Be able to write an effective README on how to use a workflow

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::: questions

- How do I split a Snakefile into manageable pieces?
- How do I avoid needing to list every file to generate in my `snakemake` call?
- What should I bear in mind when using Git for my Snakemake workflow?
- What should I include in the README for my workflow?

::::::::::::::::::::::::::::::::::::::::::::::::::

## Breaking up the Snakefile

So far,
we have written all of our rules into a single long file called `Snakefile`.
As we continue to add more rules,
this can start to get unwieldy,
and we might want to break it up into smaller pieces.

Let's do this now.
We can take the rules relating only to the `pg` output files,
and place them into a new file `workflow/rules/pg.smk`.
In their place,
in the `Snakefile`,
we add the line

```snakemake
include: "rules/pg.smk"
```

This tells Snakemake to take the contents of the new file we created,
and place it at that point in the file.
Unlike Python,
where the `import` statement creates a new _scope_,
in Snakemake,
anything defined above the `include` line is available to the included code.
So we are safe to use
the configuration parameters and metadata that we load at the top of the file.

:::::::::::::::::::::::::::::::::::::::  challenge

## Clean out the `Snakefile`

Sort the remaining rules in the `Snakefile` into additional `.smk` files.
Place these in the `rules` subdirectory with `pg.smk`.

One possible breakdown to use would be

- Rules relating to correlation function fits
- Rules relating to the Wilson flow
- Rules combining output of more than one ensemble

::::::::::::::::::::::::::::::::::::::::::::::::::


:::::::::::::::::::::::::::::::::::::::  challenge

## Tidy up the `plot_styles`

You might have noticed that
we're using the `plot_styles` configuration option
as an argument to the plotting rules,
but without including it in the `input` blocks.
Since the argument is a filename,
it is a good idea to let Snakemake know about this,
so that if we adjust the style file,
Snakemake knows to re-run the plotting rules.

Make that change now for all rules depending on `plot_styles`.

::::::::::::::::::::::::::::::::::::::::::::::::::

## A default target

When we come to run the full workflow
to generate all of the assets for our publication,
it is frustrating to need to list every single file we would like.
It is much better if we can do this as part of the workflow,
and ask Snakemake to generate a default set of targets.

We can do this by adding the following rule above all others in our `Snakefile`:


```snakemake
tables = expand("assets/tables/{table}.tex", table=["counts"])
plots = expand(
    f"assets/plots/{{plot}}{config['plot_filetype']}",
    plot=["plaquette_scan", "spectrum"],
)


rule all:
    input:
        plots=plots,
        tables=tables,
    default_target: True
```

Unlike the rules we looked at in the previous section,
this one should stay in the main Snakefile.
Note that the rule only has _inputs_:
Snakemake sees that those files must be present for the rule to complete,
so runs the rules necessary to generate them.
When we call Snakemake with no targets,
as

```shellsession
snakemake --cores all --use-conda
```

then Snakemake will look to the `all` rule,
and make `assets/tables/counts.tex`,
 `assets/plots/plaquette_scan.pdf`,
 and `assets/plots/spectrum.pdf`,
along with any intermediary files they depend on.

:::::::::::::::::::::::::::::::::::::::: callout

It can also be a good idea to add a provenance stamp at this point,
where you create an additional data file listing all outputs the workflow generated,
along with hashes of their contents,
to make it more obvious if
any files left over from previous workflow runs sneak into the output.

::::::::::::::::::::::::::::::::::::::::::::::::

## Using Snakemake with Git

Using a version control system such as [Git][git]
is good practice when developing software of any kind,
including analysis workflows.
There are some steps that we can take to make our workflows fit more nicely into Git.

Firstly,
we'd like to avoid committing files to the repository
that aren't part of the workflow definition.
We make use of the file `.gitignore` in the repository root for this.
In general,
our Git repository should only contain things that should not change from run to run,
such as the workflow definition and any workflow-specific code.
Some examples of things that probably shouldn't be committed include:

- Input data and metadata.
  These should live and be shared separately from the workflow,
  as they are inputs to the analysis
  rather than a dedicated part of the analysis workflow.
  This means that
  someone wanting to read the code doesn't need to download gigabytes of unwanted data.
- Intermediary and output files.
  You may wish to share these with readers,
  and the output plots and tables should be included in your papers.
  But if a reader wants these from the workflow,
  they should download the input data and workflow,
  and run the latter.
  Mixing generated files with the repository will confuse matters,
  and make it so that any workflow re-runs create unwanted "uncommitted changes"
  that Git will notify you about.
- `.snakemake` directory.
  Similarly to the intermediary and output files,
  this will change from machine to machine and should not be committed.
  (In particular,
  it will get quite large with Conda environments,
  which are only useful on the computer they were created on.)

:::::::::::::::::::::::::::::::::::::::: callout

If you quote data from another paper,
and have had to transcribe this into a machine-readable format like CSV yourself,
then this may be included in the repository,
since it is not a data asset
that you can reasonably publish in a data repository under your own name.
In this case,
the data's provenance and attribution should be clearly stated.

::::::::::::::::::::::::::::::::::::::::::::::::::

Let's add these to the `.gitignore` for our workflow now.

```shellsession
nano .gitignore
```

To this file,
we'll add the lines

```output
.snakemake/
raw_data/
metadata/
intermediary_data/
assets/
data_assets/
```

Someone downloading your workflow
will need to place the input data and metadata in the correct locations.
You might wish to prepare an empty directory
for the reader to place the necessary files into
Since Git does not track empty directories,
only files,
we create an empty file in it,
and update the `.gitignore` rules to un-ignore only that file.

```shellsession
touch metadata/.git_keep
nano .gitignore
```

```output
!*/.git_keep
```

Let's commit these changes now

```shellsession
git add .gitignore
git commit -m "Ignore workflow inputs, and outputs, and temporary files"
git add metadata/.git_keep
git commit -m "Prepare empty directory for metadata"
```

Where you use libraries that you have written,
that are not on the [Python Package Index][pypi],
it is a good idea to incorporate these as Git submodules in your workflow.
While `pip` can install packages from GitHub repositories,
this is not robust against you moving or closing your GitHub account later,
or GitHub stopping offering free services.
Instead,
you can make available a ZIP file containing the workflow and its primary dependencies,
which will continue to be installable significantly further into the future.

Currently we have a directory `libs/su2pg_analysis`,
containing the library we have used for most of this analysis.
This library is hosted on GitHub at `https://github.com/edbennett/su2pg_analysis`.
To track this as a Git submodule,
we use the command

```shellsession
git submodule add https://github.com/edbennett/su2pg_analysis libs/su2pg_analysis
git commit -m "Add su2pg_analysis as submodule"
```


:::::::::::::::::::::::::::::::::::::::  challenge

## What to include?

Which of the following files
should be included in a hypothetical workflow repository?
 
1. `sort.py`, a tool for putting results in the correct order to show in a paper.
3. `out_corr`, a correlation function output file.
2. `Snakefile`, the workflow definition.
4. `.snakemake/`, the Snakemake cache directory.
5. `spectrum.pdf`, a plot created by the workflow.
6. `prd.mplstyle`, a Matplotlib style file used by the workflow.
7. `README.md`, guidelines on how to use the workflow.
8. `id_rsa`, the SSH private key used to connect to clusters to run the workflow.`

:::::::::::::::  solution

## Solution

1. Yes,
   this tool is clearly part of the workflow,
   so should be included in the repository,
   unless it's being used from an external library.
2. No,
   this is input data,
   so is not part of the software.
   This should be part of a data release,
   as otherwise a reader won't be able to run the workflow,
   but should be separate to the Git repository.
3. Yes,
   our workflow release wouldn't be complete without the workflow definition.
4. No,
   this directory contains files that are specific to your computer,
   so aren't useful to anyone else.
5. No,
   this will change from run to run,
   and is not part of the software.
   A reader can regenerate it from your code and data,
   by using the workflow.
6. Yes,
   while this is input to the workflow,
   it does not change based on the input data,
   and isn't generated from the physics,
   so forms part of the workflow itself.
7. Yes,
   this is essential for a reader to be able to understand how to use the workflow,
   so should be part of the repository.
8. No,
   this is private information
   that may allow others to take over or abuse your supercomputing accounts.
   Be very careful not to put private information in public Git repositories!

:::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::

## `README`

In general,
we would like others to be able to reproduce our work&mdash;this
is a key aspect of the scientific method.
To this end,
it's important to let them know how to use our workflow,
to minimise the amount of effort that must be spent getting it running.
By convention,
this is done in a file called `README`,
(with an appropriate file extension).

In this context,
it's good to remember that "other people"
includes "your collaborators"
and "yourself in six months' time",
so writing a good README isn't just good citizenship to help others,
it's also directly  beneficial to you and those you work with.

One good format to write a README in is [Markdown][markdown].
This is rendered to formatted text automatically by most Git hosting services,
including GitHub.
When using this,
the file is conventionally called `README.md`.

Things you should include in your README include:

- A brief statement of what the workflow does,
  including a link to the paper it was written for.
- A listing of what software is required,
  including links for installation instructions or downloads.
  (Snakemake is one such piece of software.)
- Instructions on downloading the repository and necessary data,
  including a link to where the data can be found.
- Instructions on how to run the workflow,
  including the suggested `snakemake` invocation.
- An indication of the time required to run the workflow,
  and the hardware that that estimate applies to.
- Details of where output from the workflow is placed.
- An indication of how reusable the workflow is:
  is it intended to work with any set of input data,
  and has it been tested for this,
  or is it designed for the specific dataset being analysed?

Rather than writing from fresh,
you may wish to work from a template for this.
One suitable README template can be found in
[the TELOS Collaboration's workflow template][telos-template]

:::::::::::::::::::::::::::::::::::::::  challenge

## Write a README

Use the [TELOS Collaboration template][telos-template]
to draft a file `README.md` for the workflow we have developed.

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::: keypoints

- Use `.smk` files in `workflow/rules` to compartmentalise the `Snakefile`,
  and use `input:` lines in the main `Snakefile` to link them into the workflow.
- Add a rule at the top of the `Snakefile` with `default_target: True`
  to specify the default output of a workflow.
- Use `.gitignore` to avoid committing input or output data
  or the Snakemake cache.
- Use `.git_keep` files to preserve empty directories.
- Use Git submodules to link to libraries you have written that aren't on PyPI.
- Include a `README.md` in your repository explaining how to run the workflow.

::::::::::::::::::::::::::::::::::::::::::::::::::


[git]: https://git-scm.org
[markdown]: https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax
[pypi]: https://pypi.org
[telos-template]: https://github.com/telos-collaboration/workflow_template/blob/main/README.md
