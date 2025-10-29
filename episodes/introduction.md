---
title: "Running commands with Snakemake"
teaching: 20
exercises: 5
---

:::::::::::::::::::::::::::::::::::::: questions 

- How do I run a simple command with Snakemake?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Create a Snakemake recipe (a Snakefile)
- Use Snakemake to compute the average plaquette of an ensemble

::::::::::::::::::::::::::::::::::::::::::::::::

## Introduction

Data analysis in lattice quantum field theory generally has many moving parts:
you will likely have many ensembles,
with differing physical and algorithmic parameters,
and for each many different observables may be computed.
These need to be combined in different ways,
making sure that compatible statistics are used.
Making sure that each step runs in the correct order is non-trivial,
requiring careful bookkeeping,
especially if we want to update data as ensembles are extended,
if we want to take advantage of parallelism to get results faster,
and if we want auditability to be able to verify later what steps were performed.

While we could build up tools to do all of these things from scratch,
these are not challenges exclusive to lattice,
and so we can take advantage of others' work rather than reinventing the wheel.
This frees up our time to focus on the physics challenges.
The category of
"tools to help run complex arrangements of tools in the right order"
is called
"workflow management";
there are  workflow managers available,
most of which are specialised to a specific class of applications.

One workflow manager developed for scientific data analysis is called [Snakemake][snakemake];
this will be the target of this lesson.
Snakemake is similar to [GNU Make][make],
in that you create a text file containing
rules specifying how input files are translated to output files,
and then the software will work out what rules to run
to generate a specified output
from the available input files.
Unlike Make,
Snakemake uses a syntax closely based on Python,
and files containing rules can be extended using standard Python syntax.
It also has many quality-of-life improvements compared to Make,
and so is much better suited for writing data analysis workflows.

At this point,
you should have Snakemake already installed and available to you.
To test this,
we can open a terminal and run

```shellsession
$ snakemake --version
8.25.3
```

If you instead get a "command not found" error,
go back to the [setup][setup]
and check that you have completed all the necessary steps.

::::::::::::::::::::::::::::::::::: instructor

## Environment activation

The most likely issue learners will encounter here is
needing to activate their Snakemake environment when they have opened a fresh terminal.
This is hopefully as simple as 

```shellsession
conda activate snakemake
```

If Conda isn't set up to automatically activate itself on starting a shell session,
they may also need to run something like

```shellsession
source ~/miniconda3/bin/activate
```

where the exact path to run will depend on their specific setup.

:::::::::::::::::::::::::::::::::::::::

## Looking at the sample data

You should already have the sample data files unpacked.
(If not,
refer back to the [lesson setup][setup].)
Under the `su2pg/raw_data` directory,
you will find a series of subdirectories,
each containing data for a single ensemble.
In each are files containing the log of the configuration generation,
the computation of the quenched meson spectrum,
and the computation of the [Wilson flow][wilson-flow].

The sample data are for the SU(2) pure Yang-Mills theory,
and have been generated using the [HiRep][hirep] code.
We can look at their structure with `less`,
for example,
we might check the log of
generating the $\beta=2.0$ ensemble with the heat bath algorithm:

```shellsession
less raw_data/beta2.0/out_pg
```

Each log contains header lines describing the setup,
information on the computation being computed,
and results for observables computed on each configuration.
Code to parse these logs and compute statistics 
is included with the sample data;
we'll use these in due course.

To exit `less`, press <kbd>q</kbd>.

## Making a Snakefile

To start with,
let's define a rule to count the number of lines in one of the raw data files.

Within the `su2pg/workflow` directory,
edit a new text file named `Snakefile`.
Into it,
insert the following content:

```snakemake
rule count_lines:
    input: "raw_data/beta2.0/out_pg"
    output: "intermediary_data/beta2.0/pg.count"
    shell:
        "wc -l raw_data/beta2.0/out_pg > intermediary_data/beta2.0/pg.count"
```

:::::::::::::::::::::::::::::::::::::::  checklist

## Key points about this file

1. The file is named `Snakefile` - with a capital `S` and no file extension.
2. Some lines are indented. Indents must be with space characters, not tabs.
3. The rule definition starts with the keyword `rule` followed by the rule name,
   then a colon.
4. We named the rule `count_lines`.
   You may use letters, numbers or underscores,
   but the rule name
   must begin with a letter and may not be a keyword.
5. The keywords `input`, `output`, `shell` are all followed by a colon.
6. The file names and the shell command are all in `"quotes"`.
7. The file names are specified relative to the root directory of your workflow.

::::::::::::::::::::::::::::::::::::::::::::::::::

The first line tells Snakemake we are defining a new rule.
Subsequent indented lines form a part of this rule;
while there are none here,
any subsequent unindented lines would not be included in the rule.
The `input:` line tells Snakemake what files to look for to be able to run this rule.
If this file is missing
(and there is no rule to create it),
Snakemake will not consider running this rule.
The `output:` line tells Snakemake what files to expect the rule to generate.
If this file is not generated,
then Snakemake will abort the workflow with an error.
Finally,
the `shell:` block tells Snakemake what shell commands to run
to get the specified output from the given input.

Going back to the shell now,
we can test this rule.
First up,
we need to enter the directory containing the workflow

``` shellsession
cd su2pg
```

We'll spend most of the rest of the lesson in this directory.

:::::::::::::::::::::::::::::::::::::::  callout

## Activate your environment

To call `snakemake`,
we need to have active the environment that we created in the [setup](/setup).
Current versions of Conda by default prepend this environment name to your prompt,
so if you don't see `(snakemake)` in your prompt,
you will need to activate this

```shellsession
conda activate snakemake
```

::::::::::::::::::::::::::::::::::::::::::::::::

From here,
we can now run the command

```shellsession
snakemake --cores 1 --forceall --printshellcmds intermediary_data/beta2.0/pg.count
```

If we've made any transcription errors in the rule
(missing quotes, bad indentations, etc.),
then it will become clear at this point,
as we'll receive an error that we will need to fix.

:::::::::::::::::::::::::::::::::::::::  callout

Snakemake interprets all inputs and ouputs as relative to the working directory.
For this reason,
you should always run `snakemake` from the root of your workflow repository.

An option to make this easier is
to have a terminal open to the correct directory that you don't use `cd` in,
so it is always in the right place.
You can edit your workflow in a separate window,
either in another terminal with `nano` or `vim`,
or in a separate text editing application.

::::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::: instructor

## Directories

Learners less experienced with the shell
may want to `cd` into directories to edit files;
if they do this and forget to `cd` back out again,
they will encounter difficulties
as Snakemake may not be able to find the Snakefile or the input files.

If they try to work around this,
they may end up with multiple Snakefiles
or ones with inputs pointing at incorrect relative paths.

Technically,
you can specify absolute paths in Snakefiles,
but this is not recommended,
for portability reasons.
For example,
when using Snakemake to execute some rules on another machine,
this would fail as it cannot gather the dependencies into the correct location;
similarly if someone else were to run a workflow on their own machine,
the home directory is unlikely to be the same,
so the workflow would fail.

New researchers frequently like to hardcode absolute paths to their data,
so this is an important point to reinforce.

:::::::::::::::::::::::::::::::::::::::

For now,
we will consistently run `snakemake` with the ` --cores 1 --forceall --printshellcmds` options.
As we move through the lesson,
we'll explain in more detail when we need to modify them.

Let's check that the output was correctly generated:

```shellsession
$ cat intermediary_data/beta2.0/pg.count
  31064 raw_data/beta2.0/out_pg
```



:::::::::::::::::::::::::::::::::::::::  callout

You might have noticed that we are grouping files into directories
like `raw_data` and `intermediary_data`.
It is generally a good idea
to keep raw input data separate from data generated by the analysis.
This means that if you need to run a clean analysis starting from your input data,
then it is much easier to know what to remove and what to keep.
Ideally,
the `raw_data` directory should be kept read-only,
so that you don't accidentally modify your input data.
Similarly,
it is a good idea to separate out "files that you want to include in a paper"
from "intermediary files generated by the workflow but not needed in the paper";
we'll talk more about that in a later section.

You might also worry that your tooling will need
to use `mkdir` to create these directories;
in fact,
Snakemake will automatically create all directories
where it expects to see output from rules that it runs.

::::::::::::::::::::::::::::::::::::::::::::::::::


::::::::::::::::::::::::::::::::::: instructor

## Read-only data

The example data for this lesson uses read-only raw data throughout,
including the containing directories.
If learners end up with multiple copies of the data and need to delete one,
they should use the commands:

```shellsession
chmod -R u+w raw_data
rm -r raw_data
```

Having the containing directories read-only means
that extra output files can't be added by accident.
It's a relatively strict measure&mdash;while assembling data,
one would only want the files read-only,
so you could keep adding more files as they were ready.

:::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::: instructor

## Use of the `--forceall` flag

In the first few episodes we always run Snakemake with the `--forceall` flag, and it's not explained what
this does until Ep. 04. The rationale is that the default Snakemake behaviour when pruning the DAG
leads to learners seeing different output (typically the message "nothing to be done") when
repeating the exact same command. This can seem strange to learners who are used to scripting and
imperative programming.

The internal rules used by Snakemake to determine which jobs in the DAG are to be run, and which
skipped, are pretty complex, but the behaviour seen under `--forceall` is much more simple and consistent;
Snakemake simply runs every job in the DAG every time. You can think of `--forceall` as disabling the lazy
evaluation feature of Snakemake, until we are ready to properly introduce and understand it.

:::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::  challenge

## Running Snakemake

Run `snakemake --help | less` to see the help for all available options.
What does the `--printshellcmds` option in the `snakemake` command above do?

1. Protects existing output files
2. Prints the shell commands that are being run to the terminal
3. Tells Snakemake to only run one process at a time
4. Prompts the user for the correct input file

*Hint: you can search in the text by pressing <kbd>/</kbd>, and quit back to the shell with <kbd>q</kbd>*

:::::::::::::::  solution

## Solution

(2) Prints the shell commands that are being run to the terminal

This is such a useful thing we don't know why it isn't the default! The `--cores 1` option is what
tells Snakemake to only run one process at a time, and we'll stick with this for now as it
makes things simpler. The `--forceall` option tells Snakemake to always recreate output files, and
we'll learn about protected outputs much later in the course. Answer 4 is a total red herring,
as Snakemake never prompts interactively for user input.

:::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::

## Counting trajectories

The count of output lines isn't particularly useful.
Potentially more interesting is the number of trajectories in a given file.
In a HiRep generation log,
each trajectory concludes with a line of the form
```output
[MAIN][0]Trajectory #1: generated in [39.717707 sec]
```

We can use `grep` to count these,
as

```bash
grep -c generated raw_data/beta2.0/out_pg
```

:::::::::::::::::::::::::::::::::::::::  challenge

## Counting sequences in Snakemake

Modify the Snakefile to count the number of **trajectories** in `raw_data/beta2.0/out_pg`,
rather than the number of **lines**.

- Rename the rule to `count_trajectories`
- Keep the output file name the same
- Remember that the result needs to go into the output file, not just be printed on the screen
- Test the new rule once it is done.

:::::::::::::::  solution

## Solution

```snakemake
rule count_trajectories:
    input: "raw_data/beta2.0/out_pg"
    output: "intermediary_data/beta2.0/pg.count"
    shell:
        "grep -c generated raw_data/beta2.0/out_pg > intermediary_data/beta2.0/pg.count"
```

:::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::



:::::::::::::::::::::::::::::::::::::::: keypoints

- Before running Snakemake you need to write a Snakefile
- A Snakefile is a text file which defines a list of rules
- Rules have inputs, outputs, and shell commands to be run
- You tell Snakemake what file to make and it will run the shell command defined in the
  appropriate rule

::::::::::::::::::::::::::::::::::::::::::::::::::

[hirep]: https://github.com/claudiopica/HiRep
[make]: https://www.gnu.org/software/make/
[setup]: ../learners/setup.md
[snakemake]: https://snakemake.github.io/
[wilson-flow]: https://doi.org/10.48550/arXiv.1006.4518
