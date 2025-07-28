---
title: "Placeholders and wildcards"
teaching: 10
exercises: 5
---

::::::::::::::::::::::::::::::::::::::: objectives

- Use Snakemake to compute the plaquette in any file
- Understand the basic steps Snakemake goes through when running a workflow
- See how Snakemake deals with some errors

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::: questions

- How do I make a generic rule?
- How does Snakemake decide what rule to run?

::::::::::::::::::::::::::::::::::::::::::::::::::

## Making rules more generic

In the previous two episodes,
we wrote rules to count the number of generated trajectories in,
and compute the average plaquette of,
one ensemble.
As a reminder,
this was one such rule:

```snakemake
rule count_trajectories:
    input: "raw_data/beta2.0/out_pg"
    output: "intermediary_data/beta2.0/pg.count"
    shell:
        "grep -c generated raw_data/beta2.0/out_pg > intermediary_data/beta2.0/pg.count"
```


When we needed to do the same for a second ensemble,
we made a second copy of the rule,
and changed the input and output filenames.
This is obviously not scalable to large analyses:
instead,
we would like to write one rule for each type of operation we are interested in.
To do this,
we'll need to use **placeholders** and **wildcards**.
Such a generic rule might look as follows:

```snakemake
# Count number of generated trajectories for any ensemble
rule count_trajectories:
    input: "raw_data/{subdir}/out_pg"
    output: "intermediary_data/{subdir}/pg.count"
    shell:
        "grep -c generated {input} > {output}"
```

:::::::::::::::::::::::::::::::::::::::::  callout

## Comments in Snakefiles

In the above code,
the line beginning `#` is a comment line.
Hopefully you are already in the habit of adding comments to your own software.
Good comments make any code more readable,
and this is just as true with Snakefiles.

::::::::::::::::::::::::::::::::::::::::::::::::::

`{subdir}` here is an example of a **wildcard**
Wildcards are used in the `input` and `output` lines of the rule to represent parts of filenames.
Much like the `*` pattern in the shell,
the wildcard can stand in for any text in order to make up the desired filename.
As with naming your rules,
you may choose any name you like for your wildcards;
so here we used `subdir`,
since it is describing a subdirectory.
If `subdir` is set to `beta2.0`
then the new generic rule will have the same inputs and outputs as the original rule.
Using the same wildcards in the input and output
is what tells Snakemake how to match input files to output files.

If two rules use a wildcard with the same name
then Snakemake will treat them as different
entities&mdash;rules in Snakemake are self-contained in this way.

Meanwhile,
`{input}` and `{output}` are **placeholders**
Placeholders are used in the `shell` section of a rule.
Snakemake will replace them with appropriate values before running the command:
`{input}` with the full name of the input file,
and `{output}` with the full name of the output file.

If we had wanted to include
the value of the `subdir` wildcard directly in the `shell` command,
we could have used the placeholder `{wildcards.subdir}`,
but in most cases,
as here,
we just need the `{input}` and `{output}` placeholders.

Let's test this general rule now:

```shellsession
snakemake --jobs 1 --forceall --printshellcmds --use-conda intermediary_data/beta2.0/pg.count
```

As previously,
if you see errors at this point,
there is likely a problem with your Snakefile;
check that all the rules match the ones that have appeared here,
and that there aren't multiple rules with the same name.

:::::::::::::::::::::::::::::::::::::::  challenge

## General plaquette computation

Modify your Snakefile so that it can compute the average plaquette for any ensemble,
not just the ones we wrote specific rules for in the previous episode.

Test this with some of the values of $\beta$ present in the raw data.

:::::::::::::::  solution

## Solution

The replacement rule should look like:

```snakemake
# Compute average plaquette for any ensemble from its generation log
rule avg_plaquette:
    input: "raw_data/{subdir}/out_pg"
    output: "intermediary_data/{subdir}/pg.plaquette.json.gz"
    conda: "envs/analysis.yml"
    shell:
        "python -m su2pg_analysis.plaquette {input} --output_file {output}"
```

To test this,
for example:

```shellsession
snakemake --jobs 1 --forceall --printshellcmds --use-conda intermediary_data/beta1.8/pg.plaquette.json.gz
```

:::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::


:::::::::::::::::::::::::::::::::::::::  challenge

## Choosing the right wildcards

Our rule puts the sequence counts into output files named like `pg.count`.
How would you have to change the `count_trajectories` rule definition if you wanted:

1) the output file for `raw_data/beta1.8/out_hmc`
   to be `intermediary_data/beta1.8/hmc.count`?

2) the output file for `raw_data/beta1.8/mass_fun-0.63/out_hmc`
   to be `intermediary_data/beta1.8/mass_fun-0.63/hmc.count`?

3) the output file for `raw_data/beta1.8/mass_fun-0.63/out_hmc`
   to be `intermediary_data/hmc_b3.0_m-0.63.count`
   (for `raw_data/beta1.9/mass_fun-0.68/out_pg` to be
   `intermediary_data/hmc_b1.9_m-0.68.count`, etc.)?

4) the output file for `raw_data/beta1.8/mass_fun-0.63/out_hmc`
   to be `intermediary_data/hmc_m-0.63.count`
   (for `raw_data/beta1.9/mass_fun-0.68/out_pg` to be
   `intermediary_data/hmc_m-0.68.count`, etc.)?

(Assume that both pure-gauge and HMC logs tag generated trajectories the same way.
Note that input files for the latter data are not included in the sample data,
so these will not work as-is.)

:::::::::::::::  solution

## Solution

In both cases, there is no need to change the `shell` part of the rule at all.

1)
```snakemake
input: "raw_data/{subdir}/out_hmc"
output: "intermediary_data/{subdir}/hmc.count"
```

This can be done by changing only the static parts of the `input:` and `output:` lines.

2)

This in fact requires no change from the previous answer.
The wildcard `{subdir}` can include `/`,
so can represent multiple levels of subdirectory.

3)
```snakemake
input: "raw_data/beta{beta}/mass_fun{mass}/out_hmc"
output: "intermediary_data/hmc_b{beta}_m{mass}.count"
```

In this case,
it was necessary to change the wildcards,
because the subdirectory name needs to be split
to obtain the values of $\beta$ and $m_{\mathrm{fun}}$.
The names chosen here are `{beta}` and `{mass}`,
but you could choose any names,
as long as they match between the `input` and `output` parts.

4)

This one **isn't possible**,
because Snakemake cannot determine which input file you want to count
by matching wildcards on the file name `intermediary_data/hmc_m-0.63.count`.
You could try a rule
like this:

```snakemake
input: "raw_data/beta1.8/mass_fun{mass}/out_hmc"
output: "intermediary_data/hmc_m{mass}.count"
```

...but it only works because $\beta$ is hard-coded into the `input` line,
and the rule will only work on this specific sample,
not other cases where other values of $\beta$ may be wanted.
In general,
input and output filenames need to be carefully chosen
so that Snakemake can match everything up
and determine the right input from the output filename.

:::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::::  callout

## Filenames aren't data

Notice that in some examples we can pull out the value of $\beta$
from the name of the directory in which the file is located.
However,
ideally,
we should avoid relying on this being correct.
The name and location are useful for us to find the correct file,
but we should try to ensure that the file contents also contain these data,
and that we make use of those data in preference to the filename.

::::::::::::::::::::::::::::::::::::::::::::::::::

## Snakemake order of operations

We're only just getting started with some simple rules, but it's worth thinking about exactly what
Snakemake is doing when you run it. There are three distinct phases:

1. Prepares to run:
    1. Reads in all the rule definitions from the Snakefile
2. Plans what to do:
    1. Sees what file(s) you are asking it to make
    2. Looks for a matching rule by looking at the `output`s of all the rules it knows
    3. Fills in the wildcards to work out the `input` for this rule
    4. Checks that this input file is actually available
3. Runs the steps:
    1. Creates the directory for the output file, if needed
    2. Removes the old output file if it is already there
    3. Only then, runs the shell command with the placeholders replaced
    4. Checks that the command ran without errors *and* made the new output file as expected

For example, if we now ask Snakemake to generate a file named `intermediary_data/wibble_1/pg.count`:

```output
$ snakemake --jobs 1 --forceall --printshellcmds intermediary_data/wibble_1/pg.count
Building DAG of jobs...
MissingInputException in line 1 of /home/zenmaster/data/su2pg/workflow/Snakefile:
Missing input files for rule count_trajectories:
    output: intermediary_data/wibble_1/pg.count
    wildcards: subdir=wibble_1
    affected files:
        raw_data/wibble_1/out_pg
```

Snakemake sees that 
a file with a name like this could be produced by the `count_trajectories` rule.
However, 
when it performs the wildcard substitution
it sees that the input file would need to be named `raw_data/wibble_1/out_pg`,
and there is no such file available.
Therefore Snakemake stops and gives an error before any shell commands are run.

:::::::::::::::::::::::::::::::::::::::::  callout

## Dry-run (`--dry-run`) mode

It's often useful to run just the first two phases,
so that Snakemake will plan out the jobs to run,
and print them to the screen,
but never actually run them.
This is done with the `--dry-run`
flag, eg:

```bash
$ snakemake --dry-run --forceall --printshellcmds temp33_1_1.fq.count
```

We'll make use of this later in the lesson.

::::::::::::::::::::::::::::::::::::::::::::::::::

The amount of checking may seem pedantic right now,
but as the workflow gains more steps this will become very useful to us indeed.

<!-- maybe add an extra challenge here? -->

:::::::::::::::::::::::::::::::::::::::: keypoints

- Snakemake rules are made generic with placeholders and wildcards
- Snakemake chooses the appropriate rule
  by replacing wildcards such the the output matches the target
- Placeholders in the shell part of the rule are
  replaced with values based on the chosen wildcards
- Snakemake checks for various error conditions and will stop if it sees a problem

::::::::::::::::::::::::::::::::::::::::::::::::::
