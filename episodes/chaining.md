---
title: Chaining rules
teaching: 30
exercises: 20
---

::::::::::::::::::::::::::::::::::::::: objectives

- Use Snakemake to compute and then plot the average plaquettes of multiple ensembles
- Understand how rules are linked by filename patterns
- Be able to use multiple input files in one rule

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::: questions

- How do I combine rules into a workflow?
- How can I make a rule with multiple input files?
- How should I refer to multiple files with similar names?

::::::::::::::::::::::::::::::::::::::::::::::::::

## A pipeline of multiple rules

We have so far been able to count the number of generated trajectories,
and compute the average plaquette,
given an output log from the configuration generation.
However,
an individual average plaquette is not interesting in isolation;
what is more interesting is
how it varies between different values of the input parameters.
To do this,
we will need to take the output of the `avg_plaquette` rule
that we defined earlier,
and use it as input for another rule.

Let's define that rule now:

```snakemake
# Take individual data files for average plaquette and plot combined results
rule plot_avg_plaquette:
    input:
        "intermediary_data/beta1.8/pg.plaquette.json.gz",
        "intermediary_data/beta2.0/pg.plaquette.json.gz",
        "intermediary_data/beta2.2/pg.plaquette.json.gz",
    output:
        "assets/plots/plaquette_scan.pdf"
    conda: "envs/analysis.yml"
    shell:
        "python src/plot_plaquette.py {input} --output_filename {output}"
```

:::::::::::::::::::::::::::::::::::::::  callout

You can see that here we're putting
"files that want to be included in a paper"
in an `assets` directory,
similarly to the `raw_data` and `intermediary_data` directories
we discussed in a previous episode.
It can be useful to further distinguish
plots,
tables,
and other definitions,
by using subdirectories in this directory.

::::::::::::::::::::::::::::::::::::::::::::::::::

Rather than one input,
as we have seen in rules so far,
this rule requires three.
When Snakemake substitutes these into the `{input}` placeholder,
it will automatically add a space between them.
Let's test this now:

```shellsession
snakemake --cores 1 --forceall --printshellcmds --use-conda assets/plots/plaquette_scan.pdf
```

Look at the logging messages that Snakemake prints in the terminal. What has happened here?

1. Snakemake looks for a rule to make `assets/plots/plaquette_scan.pdf`
2. It determines that the `plot_avg_plaquette` rule can do this,
   if it has `intermediary_data/beta1.8/pg.plaquette.json.gz`,
   `intermediary_data/beta2.0/pg.plaquette.json.gz`,
   and `intermediary_data/beta2.2/pg.plaquette.json.gz`.
3. Snakemake looks for a rule to make `intermediary_data/beta1.8/pg.plaquette.json.gz`
4. It determines that `avg_plaquette` can make this if `subdir=beta1.8`
5. It sees that the input needed is therefore `raw_data/beta1.8/out_pg`
6. Now Snakemake has reached an available input file,
   it runs the `avg_plaquette` rule.
7. It then looks through the other two $\beta$ values in turn,
   repeating the process until it has all of the needed inputs.
8. Finally, it runs the `plot_avg_plaquette` rule.

**Here's a visual representation of this process:**

![][fig-chaining]

This,
in a nutshell,
is how we build workflows in Snakemake.

1. Define rules for all the processing steps
2. Choose `input` and `output` naming patterns that allow Snakemake to link the rules
3. Tell Snakemake to generate the final output files

If you are used to writing regular scripts this takes a little getting used to.
Rather than listing steps in order of execution,
you are always **working backwards** from the final desired result.
The order of operations is determined by 
applying the pattern matching rules to the filenames,
not by the order of the rules in the Snakefile.

:::::::::::::::::::::::::::::::::::::::::  callout

## Choosing file name patterns

Chaining rules in Snakemake is a matter of choosing filename patterns that connect the rules.
There's something of an art to it, and most times there are several options that will work, but
in all cases the file names you choose will need to be consistent and unabiguous.

::::::::::::::::::::::::::::::::::::::::::::::::::

## Making file lists easier

In the rule above,
we plotted the average plaquette for three values of $\beta$
by listing the files expected to contain their values.
In fact,
we have data for a larger number of $\beta$ values,
but typing out each file by hand would be quite cumbersome.
We can make use of the `expand()` function to do this more neatly:

```snakemake
    input:
        expand(
            "intermediary_data/beta{beta}/pg.plaquette.json.gz",
            beta=[1.5, 1.6, 1.7, 1.8, 1.9, 2.0, 2.1, 2.2, 2.3, 2.4, 2.5],
        ),
```

The first argument to `expand()` here is a template for the filename,
and subsequent keyword arguments are
lists of variables to fill into the placeholders.
The output is the cartesian product of all the parameter lists.

We can check that this works correctly:

```shellsession
snakemake --cores 1 --forceall --printshellcmds --use-conda assets/plots/plaquette_scan.pdf
```


:::::::::::::::::::::::::::::::::::::::  challenge

## Tabulating trajectory counts

The script `src/tabulate_counts.py` will take a list of files containing plaquette data,
and output a LaTeX table of trajectory counts.
Write a rule to generate this table for all values of $\beta$,
and output it to `assets/tables/counts.tex`.

:::::::::::::::  solution

## Solution 1

The rule should look like:

```snakemake
# Output a LaTeX table of trajectory counts
rule tabulate_counts:
    input:
        expand(
            "intermediary_data/beta{beta}/pg.count",
            beta=[1.5, 1.6, 1.7, 1.8, 1.9, 2.0, 2.1, 2.2, 2.3, 2.4, 2.5],
        )
    output: "assets/tables/counts.tex"
    conda: "envs/analysis.yml"
    shell:
        "python src/tabulate_counts.py {input} > {output}"
```

To test this,
for example:

```shellsession
snakemake --cores 1 --forceall --printshellcmds --use-conda assets/tables/counts.tex
```

:::::::::::::::::::::::::

This setup currently requires reading the value of $\beta$ from the filename.
Why is this not ideal?
How would the workflow need to be changed to avoid this?

:::::::::::::::  solution

## Solution 2

It's easy for files to be misnamed when creating or copying them.
Putting the wrong data into the file is harder,
especially when it's
a raw data file generated by the same program as the rest of the data.
(If the wrong value were given as input,
this could happen,
but the corresponding output data would also be generated at that incorrect value.
Provided the values are treated consistently,
the downstream analysis could in fact still be valid,
just not exactly as intended.)

Currently,
`grep -c` is used to count the number of trajectories.
This would need to be replaced or supplemented
with a tool that read out the value of $\beta$ from the input log,
and outputs it along with the trajectory count.
The `src/tabulate_counts.py` script could then be updated to use this number,
rather than the filename.

In fact,
the `plaquette` module does just this;
in addition to the average plaquette,
it also records the number of trajectories generated
as part of the metadata and provenance information it tracks.

:::::::::::::::::::::::::


::::::::::::::::::::::::::::::::::::::::::::::::::


[fig-chaining]: fig/chaining_rules.svg {alt='A visual representation of the above process
showing the rule definitions,
with arrows added to indicate the order wildcards and placeholders are substituted.
Blue arrows start from the input of the `plot_avg_plaquette` rule,
which are the files
`intermediary_data/beta1.8/pg.plaquette.json.gz`,
`intermediary_data/beta2.0/pg.plaquette.json.gz`,
and `intermediary_data/beta2.2/pg.plaquette.json.gz`,
then point down from components of the filename to wildcards
in the output of the `avg_plaquette` rule.
Orange arrows then track back up through the shell parts of both rules, where the placeholders are,
and finally back to the target output filename at the top.'}


:::::::::::::::::::::::::::::::::::::::: keypoints

- Snakemake links up rules by iteratively looking for rules that make missing inputs
- Careful choice of filenames allows this to work
- Rules may have multiple named input files (and output files)
- Use `expand()` to generate lists of filenames from a template

::::::::::::::::::::::::::::::::::::::::::::::::::
