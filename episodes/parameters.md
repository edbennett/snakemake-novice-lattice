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
The `plot_plaquette.py` script accepts a `--style` argument,
to tell it what style file to use to plot.
One way to make use of this would be to add
`--style styles/paper.mplstyle` directly to the `shell:` block.
However,
if we had many such rules,
and wanted to switch from generating output for a paper
to generating it for a poster,
then we would need to change the value in many places.
Instead,
we can define a variable at the top of the Snakefile

```snakefile
plot_style = "styles/paper.mplstyle"
```

Then,
when we use a script to generate a plot,
we can update the `shell:` block of the corresponding rule similarly to

```
"python src/plot_plaquette.py {input} --plot_filename {output}" --styles {plot_styles}
```

Snakemake will 
substitute the value of the global `plot_styles` variable
in place of the `{plot_styles}` placeholder.`
