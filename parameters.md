---
title: "Metadata and parameters"
teaching: 10
exercises: 5
---

## Global parameters

Thus far,
each of our rules has taken one or more input files,
and given output files solely based on that.
However,
in some cases we may want to control options without having them within an input file.

For example,
in the previous episode,
we wrote a rule to plot a graph
using the script `src/plot_plaquette.py`.
The style of output we got was good for a paper,
but if we were producing a poster,
or putting the plot onto a slide with a dark background,
we may wish to use a different output style.
The `plot_plaquette.py` script accepts a `--styles` argument,
to tell it what style file to use to plot.
One way to make use of this would be to add
`--styles styles/paper.mplstyle` directly to the `shell:` block.
However,
if we had many such rules,
and wanted to switch from generating output for a paper
to generating it for a poster,
then we would need to change the value in many places.
Instead,
we can define a variable at the top of the Snakefile

```snakefile
plot_styles = "styles/jhep.mplstyle"
```

Then,
when we use a script to generate a plot,
we can update the `shell:` block of the corresponding rule similarly to

```
"python src/plot_plaquette.py {input} --output_filename {output} --plot_styles {plot_styles}"
```

Snakemake will 
substitute the value of the global `plot_styles` variable
in place of the `{plot_styles}` placeholder.

We can test this by changing `paper` to `poster`,
and running

```shellsession
snakemake --jobs 1 --forceall --printshellcmds --use-conda assets/plots/plaquette_scan.pdf
```

We can see that the generated file now uses a different set of fonts.

:::::::::::::::::::::::::::::::::::::::  challenge

## Wilson flow

The tool `su2pg_analysis.w0` computes the scale $w_0$ given
a log of the energy density during evolution of the Wilson flow for an ensemble.
To do this,
the reference scale $\mathcal{W}_0$ needs to be passed to the `--W0` parameter.
Use this,
and the logs stored in the files `out_wflow` for each ensemble's raw data directory,
to output the $w_0$ scale in a file `wflow.w0.json` for each ensemble,
taking the reference value $\mathcal{W_0} = 0.2$.

:::::::::::::::  solution

```snakemake
W0_reference = 0.2

# Compute w0 scale for single ensemble for fixed reference scale
rule w0:
    input: "raw_data/{subdir}/out_wflow"
    output: "intermediary_data/{subdir}/wflow.w0.json.gz"
    conda: "envs/analysis.yml"
    shell:
        "python -m su2pg_analysis.w0 {input} --W0 {W0_reference} --output_file {output}"
```

:::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::::  challenge

## Generating different filetypes

In addition to different plot styles,
we may also wish to generate different filetypes.
PDF is useful for including in LaTeX,
but SVG may be a better format to use with some tools.

If we add a global definition:

```snakemake
plot_filetype = "pdf"
```

and update the `output:` block of the rule as:

```snakemake
    output:
        "assets/plots/plaquette_scan.{plot_filetype}"
```

does this have the same effect as the example with `--styles` above?

(Hint:
what happens when you try to make the targets
`assets/plots/plaquette_scan.svg`
and `assets/plots/plaquette_scan.txt`
by specifying them at the command line,
without changing the value of `plot_filetype`?)

:::::::::::::::  solution

## Solution

This can achieve a similar result,
but in a slightly different way.
In the `--styles` example,
the `{plot_styles}` string is in the `shell:` block,
and so directly looks up the `plot_styles` variable.
(Recall that to look up a wildcard,
we needed to explicitly use `wildcards.`.)

However,
in this case the `{plot_filetype}` string is in the `output:` block,
so defines a wildcard.
This may take any value,
so if we instruct `snakemake` to produce `plaquette_scan.txt`,
it will diligently pass that filename to `plot_plaquette.py`.

The `plot_filetype = "pdf"` is in fact ignored.
It could however be used to set a default set of targets to generate,
which we will talk about in a later episode.

:::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::

## Metadata from a file

We would frequently like our rules to depend
on data that are specific to the specific ensembles being analysed.
For example,
consider the rule:

```snakemake
# Compute pseudoscalar mass and amplitude with fixed plateau
rule ps_mass:
    input: "raw_data/{subdir}/out_corr"
    output: "intermediary_data/{subdir}/corr.ps_mass.json.gz"
    conda: "envs/analysis.yml"
    shell:
        "python -m su2pg_analysis.meson_mass {input} --output_file {output} --plateau_start 18 --plateau_end 23"
```

This rule hardcodes the positions of the start and end of the plateau region.
In most studies,
each ensemble and observable may have a different plateau position,
so there is no good value to hardcode this to.
Instead,
we'd like a way of picking the right value from
some list of parameters that we specify.

We could do this within the Snakefile,
but where possible it is good to avoid mixing data with code.
We shouldn't need to modify our code
every time we add or modify the data it is analysing.
Instead,
we'd like to have a dedicated file containing these parameters,
and to be able to have Snakemake read it and pick out the correct values.

To do this,
we can exploit the fact that Snakemake is an extension of Python.
In particular,
Snakemake makes use of the [Pandas][pandas] library for tabular data,
which we can use to read in a CSV files.
Let's add the following to the top of the file:

```snakemake
import pandas

metadata = pandas.read_csv("metadata/ensemble_metadata.csv")
```

The file being read here is a CSV
(Comma Separated Values)
file.
We can create, view, and modify this with the spreadsheet tool of our choice.
Let's take a look at the file now.

![Screenshot of a spreadsheet application showing the file `metadata/ensemble_metadata.csv`.](./images/metadata_spreadsheet.png)

You can see that we have columns defining metadata to identify each ensemble,
and columns for parameters relating to the analysis of each ensemble.

Now,
how do we tell Snakemake to pull out the correct value from this?

```snakemake
# Compute pseudoscalar mass and amplitude, read plateau from metadata
rule ps_mass:
    input: "raw_data/beta{beta}/out_corr"
    output: "intermediary_data/beta{beta}/corr.ps_mass.json.gz"
    params:
        plateau_start=lookup(within=metadata, query="beta == {beta}", cols="ps_plateau_start"),
        plateau_end=lookup(within=metadata, query="beta == {beta}", cols="ps_plateau_end"),
    conda: "envs/analysis.yml"
    shell:
        "python -m su2pg_analysis.meson_mass {input} --output_file {output} --plateau_start {params.plateau_start} --plateau_end {params.plateau_end}"
```

We've done a couple of things here.
Firstly,
we've made explicit the reference to $\beta$ in the file paths,
so that we can use `beta` as a wildcard,
similarly to in the challenge in the previous episode.
Secondly,
we've introduced a `params:` block.
This is how we tell Snakemake about quantities that may vary from run to run,
but that are not filenames.
Thirdly,
we've used the `lookup()` function to
search the `metadata` dataframe for the ensemble that we are considering.
Finally,
we've used `{params.plateau_start}` and `{params.plateau_end}` placeholders
to use these parameters in the shell command that gets run.

Let's test this now:

```shellsession
snakemake --jobs 1 --forceall --printshellcmds --use-conda intermediary_data/beta2.0/corr.ps_mass.json
cat intermediary_data/beta2.0/corr.ps_mass.json
```

```output
TODO
```

TODO CHALLENGE





[pandas]: https://pandas.pydata.org
